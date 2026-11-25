# Configuration changes, step by step

Start from the configurations in `configs/`. Each step below lists the change to commit, in Junos
configuration format, and what to check afterwards. `# was:` comments show the value being replaced.

All referenced policies are already defined in the configurations. A step only changes which policy
is applied.

## Step 1: Maintenance begins (AWS Router 1)

AWS Router 1 keeps its BGP sessions up but makes its path less attractive in both directions.

- **Toward the customer**, every prefix in every VRF carries all three signals: AS path prepended
  three times, community 7224:2200 and community graceful-shutdown (65535:0). Each VRF's customer
  policy reacts to one of them.
- **Toward AWS**, routes learned from the customer get local-preference 50, below the backup path.

Change on AWS Router 1. The same change applies to each VRF and its Customer Edge 1 neighbor:

| Scenario | VRF | Neighbor |
|---|---|---|
| 01 | PREPEND | 10.1.1.1 |
| 02 | COMMUNITY | 10.1.3.1 |
| 03 | GSHUT | 10.1.2.1 |

```
routing-instances {
    GSHUT {
        protocols {
            bgp {
                group customer-edge-1 {
                    neighbor 10.1.2.1 {
                        import IMPORT-FROM-CUSTOMER-MAINT;   # was: IMPORT-FROM-CUSTOMER-NORMAL
                        export EXPORT-MAINT-ALL;             # was: EXPORT-TO-CUSTOMER-NORMAL
                    }
                }
            }
        }
    }
}
```

Policies used (from `configs/aws-router-1.conf`):

```
policy-options {
    community DX-MAINTENANCE members 7224:2200;
    community GRACEFUL_SHUTDOWN members 65535:0;
    policy-statement EXPORT-MAINT-ALL {
        term all-signals {
            from protocol bgp;
            then {
                as-path-prepend "7224 7224 7224";
                community add DX-MAINTENANCE;
                community add GRACEFUL_SHUTDOWN;
                accept;
            }
        }
    }
    policy-statement IMPORT-FROM-CUSTOMER-MAINT {
        term deprioritize {
            then {
                local-preference 50;
                accept;
            }
        }
    }
}
```

How each customer VRF reacts on Customer Edge 1 (policies in `configs/customer-edge-1.conf`):

| VRF | Customer Edge 1 import policy | Path via AWS Router 1 after the change | Why Customer Network now prefers Customer Edge 2 |
|---|---|---|---|
| PREPEND | none | AS path `7224 7224 7224 7224`, local-preference 100 | Longer AS path |
| COMMUNITY | `DX-MAINT-PREPEND` | AS path `65001 65001 65001 65001 65001 7224 7224 7224 7224` | Longer AS path |
| GSHUT | `GSHUT-IMPORT` | local-preference 0 | Lower local-preference |

In COMMUNITY, Customer Edge 2 advertises on-premises with six prepends of 65001, so AWS's three
prepends alone are not enough to move traffic. The customer's own five prepends on Customer Edge 1 are
what move it.

Check:

- Customer Network: in every VRF the active path to the VPC prefix is via Customer Edge 2 (10.22.x.1).
- DXGW: in every VRF the active path to 10.0.0.0/8 is via AWS Router 2 (10.10.2.1).
- AWS Router 1: `show route advertising-protocol bgp 10.1.2.1 table GSHUT.inet.0 detail` shows the
  prepended AS path and both communities.

## Step 2: Stop advertising (AWS Router 1)

AWS Router 1 withdraws its routes in both directions. It still accepts routes, so it can forward
traffic that is still arriving.

Change on AWS Router 1, for each VRF and its Customer Edge 1 neighbor (same table as Step 1):

```
routing-instances {
    GSHUT {
        vrf-export EXPORT-REJECT-ALL;                        # new
        protocols {
            bgp {
                group customer-edge-1 {
                    neighbor 10.1.2.1 {
                        export EXPORT-REJECT-ALL;            # was: EXPORT-MAINT-ALL
                    }
                }
            }
        }
    }
}
```

```
policy-options {
    policy-statement EXPORT-REJECT-ALL {
        term reject-all {
            then reject;
        }
    }
}
```

Check: traffic stays on Customer Edge 2 and AWS Router 2 in every VRF. AWS Router 1's routes no
longer appear on Customer Edge 1 or DXGW.

## Step 3: Return to service, part 1: advertise again as the less preferred path (AWS Router 1)

The reverse of Step 2. AWS Router 1 advertises its routes again in both directions, but keeps the
maintenance signals and the lowered local-preference from Step 1. Its paths come back as backup
paths, so BGP converges on them while traffic stays on AWS Router 2 and Customer Edge 2.

Change on AWS Router 1, for each VRF and its Customer Edge 1 neighbor (same table as Step 1):

```
routing-instances {
    GSHUT {
        delete: vrf-export;                                  # was: EXPORT-REJECT-ALL
        protocols {
            bgp {
                group customer-edge-1 {
                    neighbor 10.1.2.1 {
                        export EXPORT-MAINT-ALL;             # was: EXPORT-REJECT-ALL
                        # import stays IMPORT-FROM-CUSTOMER-MAINT
                    }
                }
            }
        }
    }
}
```

Check:

- Customer Edge 1 has the VPC prefix from AWS Router 1 again, with the Step 1 signals.
- DXGW has two paths to 10.0.0.0/8 in every VRF, and the active one is still via AWS Router 2
  (10.10.2.1).
- The Customer Network still prefers Customer Edge 2 (10.22.x.1) in every VRF.

Wait here until both paths are present and stable before going on.

## Step 4: Return to service, part 2: remove the maintenance signals (AWS Router 1)

The reverse of Step 1. AWS Router 1 returns to its steady-state policies, and traffic moves back to
it in both directions.

Change on AWS Router 1, for each VRF and its Customer Edge 1 neighbor:

```
routing-instances {
    GSHUT {
        protocols {
            bgp {
                group customer-edge-1 {
                    neighbor 10.1.2.1 {
                        import IMPORT-FROM-CUSTOMER-NORMAL;  # was: IMPORT-FROM-CUSTOMER-MAINT
                        export EXPORT-TO-CUSTOMER-NORMAL;    # was: EXPORT-MAINT-ALL
                    }
                }
            }
        }
    }
}
```

Check: in every VRF the Customer Network prefers Customer Edge 1 (10.21.x.1) and DXGW prefers AWS
Router 1 (10.10.1.1) again.

## Step 5: Customer-initiated failover (Customer Edge 2)

The customer promotes the backup connection to primary. Only Customer Edge 2 changes. AWS and
Customer Edge 1 are not touched.

- **Toward AWS**, in every VRF, Customer Edge 2 tags on-premises routes with 7224:7300 (high
  preference) instead of 7224:7100 (low preference).
- **Toward on-premises**:
  - In GSHUT, Customer Edge 2 sets local-preference 200 on routes from AWS Router 2,
    above Customer Edge 1's 150.
  - In PREPEND and COMMUNITY, Customer Edge 2 stops prepending toward the Customer Network.

Change on Customer Edge 2:

```
routing-instances {
    PREPEND {
        protocols {
            bgp {
                group aws-router-2 {
                    neighbor 10.2.1.2 {
                        export CUSTOMER-EXPORT-FAILOVER;     # was: CUSTOMER-EXPORT-BACKUP
                    }
                }
                group customer-network {
                    neighbor 10.22.1.2 {
                        delete: export;                      # was: ONPREM-PREPEND-3X
                    }
                }
            }
        }
    }
    GSHUT {
        protocols {
            bgp {
                group aws-router-2 {
                    neighbor 10.2.2.2 {
                        import PREFER-BACKUP;                # new (no import policy before)
                        export CUSTOMER-EXPORT-FAILOVER;     # was: CUSTOMER-EXPORT-BACKUP
                    }
                }
            }
        }
    }
    COMMUNITY {
        protocols {
            bgp {
                group aws-router-2 {
                    neighbor 10.2.3.2 {
                        export CUSTOMER-EXPORT-FAILOVER;     # was: CUSTOMER-EXPORT-BACKUP
                    }
                }
                group customer-network {
                    neighbor 10.22.3.2 {
                        delete: export;                      # was: ONPREM-PREPEND-6X
                    }
                }
            }
        }
    }
}
```

Policies used (from `configs/customer-edge-2.conf`):

```
policy-options {
    community DX-HIGH-PREF members 7224:7300;
    policy-statement CUSTOMER-EXPORT-FAILOVER {
        term on-prem {
            from protocol bgp;
            then {
                community add DX-HIGH-PREF;
                accept;
            }
        }
    }
    policy-statement PREFER-BACKUP {
        term prefer {
            then {
                local-preference 200;
                accept;
            }
        }
    }
}
```

Check: in every VRF the Customer Network prefers Customer Edge 2 (10.22.x.1) and DXGW prefers AWS
Router 2 (10.10.2.1).

## Step 6: Customer restore (Customer Edge 2)

Reverse Step 5 on Customer Edge 2:

- Apply `CUSTOMER-EXPORT-BACKUP` as the export toward AWS Router 2 in all three VRFs.
- Remove the `PREFER-BACKUP` import in GSHUT.
- Apply `ONPREM-PREPEND-3X` (PREPEND) and `ONPREM-PREPEND-6X` (COMMUNITY) as the export toward the
  Customer Network.

Check: in every VRF the Customer Network prefers Customer Edge 1 and DXGW prefers AWS Router 1.
