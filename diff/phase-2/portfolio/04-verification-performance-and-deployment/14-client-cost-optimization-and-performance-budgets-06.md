## `chore(perf): 최종 lab 성능 측정 결과 기록`

diff --git a/performance/lighthouse-baseline.json b/performance/lighthouse-baseline.json
new file mode 100644
index 0000000..e0346bc
--- /dev/null
+++ b/performance/lighthouse-baseline.json
@@ -0,0 +1,348 @@
+{
+  "schemaVersion": 1,
+  "measuredAt": "2026-08-13T02:52:54.086Z",
+  "measurement": {
+    "kind": "Lighthouse desktop lab run against the local production server",
+    "runCountPerUrl": 3,
+    "aggregation": "median",
+    "interactionProxy": "total-blocking-time"
+  },
+  "targets": {
+    "accessibilityScore": 95,
+    "cls": 0.1,
+    "lcpMs": 2500,
+    "performanceScore": 90,
+    "tbtMs": 200
+  },
+  "environment": {
+    "arch": "arm64",
+    "chromeUserAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/147.0.0.0 Safari/537.36",
+    "cpu": "Apple M1",
+    "logicalCpuCount": 8,
+    "memoryBytes": 17179869184,
+    "node": "v24.18.0",
+    "platform": "darwin"
+  },
+  "routes": {
+    "http://localhost:3300/?view=brutalist": {
+      "median": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 601.511,
+        "performanceScore": 100,
+        "tbtMs": 0
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 602.624,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 601.511,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 586.88,
+          "performanceScore": 100,
+          "tbtMs": 0
+        }
+      ]
+    },
+    "http://localhost:3300/?view=cinematic": {
+      "median": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 609.4290000000001,
+        "performanceScore": 100,
+        "tbtMs": 0
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 647.3175000000001,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 609.4290000000001,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 596.3915999999999,
+          "performanceScore": 100,
+          "tbtMs": 0
+        }
+      ]
+    },
+    "http://localhost:3300/?view=classic": {
+      "median": {
+        "accessibilityScore": 100,
+        "cls": 0.000006999999999999999,
+        "lcpMs": 653.328,
+        "performanceScore": 100,
+        "tbtMs": 0
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0.000009,
+          "lcpMs": 659.009,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0.000006999999999999999,
+          "lcpMs": 633.0525,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0.000005999999999999999,
+          "lcpMs": 653.328,
+          "performanceScore": 100,
+          "tbtMs": 0
+        }
+      ]
+    },
+    "http://localhost:3300/?view=design": {
+      "median": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 592.8209999999999,
+        "performanceScore": 100,
+        "tbtMs": 0
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 612.711,
+          "performanceScore": 99,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 592.8209999999999,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 571.1610000000001,
+          "performanceScore": 100,
+          "tbtMs": 0
+        }
+      ]
+    },
+    "http://localhost:3300/?view=editorial": {
+      "median": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 541.8625,
+        "performanceScore": 100,
+        "tbtMs": 0
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 551.172,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 533.933,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 541.8625,
+          "performanceScore": 100,
+          "tbtMs": 0
+        }
+      ]
+    },
+    "http://localhost:3300/projects/example-project?view=brutalist": {
+      "median": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 548.85625,
+        "performanceScore": 100,
+        "tbtMs": 0
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 592.3775,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 457.45400000000006,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 548.85625,
+          "performanceScore": 100,
+          "tbtMs": 0
+        }
+      ]
+    },
+    "http://localhost:3300/projects/example-project?view=cinematic": {
+      "median": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 616.348,
+        "performanceScore": 100,
+        "tbtMs": 0
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 609.8167,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 699.694,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 616.348,
+          "performanceScore": 100,
+          "tbtMs": 0
+        }
+      ]
+    },
+    "http://localhost:3300/projects/example-project?view=classic": {
+      "median": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 564.735,
+        "performanceScore": 100,
+        "tbtMs": 0
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 508.326,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 573.335,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 564.735,
+          "performanceScore": 100,
+          "tbtMs": 0
+        }
+      ]
+    },
+    "http://localhost:3300/projects/example-project?view=design": {
+      "median": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 563.3549999999999,
+        "performanceScore": 100,
+        "tbtMs": 0
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 569.877,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 554.1369000000001,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 563.3549999999999,
+          "performanceScore": 100,
+          "tbtMs": 0
+        }
+      ]
+    },
+    "http://localhost:3300/projects/example-project?view=editorial": {
+      "median": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 555.6975,
+        "performanceScore": 100,
+        "tbtMs": 0
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 626.245,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 555.6975,
+          "performanceScore": 100,
+          "tbtMs": 0
+        },
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 533.558,
+          "performanceScore": 100,
+          "tbtMs": 0
+        }
+      ]
+    }
+  }
+}
diff --git a/performance/lighthouse-mobile-observation.json b/performance/lighthouse-mobile-observation.json
new file mode 100644
index 0000000..d068c24
--- /dev/null
+++ b/performance/lighthouse-mobile-observation.json
@@ -0,0 +1,244 @@
+{
+  "schemaVersion": 1,
+  "measuredAt": "2026-08-13T02:57:31.947Z",
+  "measurement": {
+    "kind": "Lighthouse mobile lab run against the local production server",
+    "enforcement": "observation-only",
+    "runCountPerUrl": 1,
+    "aggregation": "single idle run",
+    "interactionProxy": "total-blocking-time",
+    "sourceReports": ".lighthouseci/lhr-*.json"
+  },
+  "targets": {
+    "accessibilityScore": 95,
+    "cls": 0.1,
+    "lcpMs": 2500,
+    "performanceScore": 90,
+    "tbtMs": 200
+  },
+  "environment": {
+    "arch": "arm64",
+    "chromeUserAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/147.0.0.0 Safari/537.36",
+    "cpu": "Apple M1",
+    "lighthouse": "12.6.1",
+    "logicalCpuCount": 8,
+    "memoryBytes": 17179869184,
+    "node": "v24.18.0",
+    "platform": "darwin"
+  },
+  "summary": {
+    "routeCount": 10,
+    "passedTargets": {
+      "performanceScore": 9,
+      "accessibilityScore": 10,
+      "lcp": 3,
+      "tbt": 9,
+      "cls": 10
+    },
+    "average": {
+      "performanceScore": 94.2,
+      "lcpMs": 2563.0351500000006,
+      "tbtMs": 162.23589999999996
+    },
+    "range": {
+      "performanceScore": [
+        89,
+        98
+      ],
+      "lcpMs": [
+        1840.478,
+        3198.845
+      ],
+      "tbtMs": [
+        123.09175000000005,
+        218.93099999999936
+      ],
+      "cls": [
+        0,
+        0.00007099999999999999
+      ]
+    }
+  },
+  "routes": {
+    "http://localhost:3300/?view=brutalist": {
+      "observation": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 2791.8460000000005,
+        "performanceScore": 93,
+        "tbtMs": 156
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 2791.8460000000005,
+          "performanceScore": 93,
+          "tbtMs": 156
+        }
+      ]
+    },
+    "http://localhost:3300/?view=cinematic": {
+      "observation": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 2900.4494999999997,
+        "performanceScore": 94,
+        "tbtMs": 123.09175000000005
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 2900.4494999999997,
+          "performanceScore": 94,
+          "tbtMs": 123.09175000000005
+        }
+      ]
+    },
+    "http://localhost:3300/?view=classic": {
+      "observation": {
+        "accessibilityScore": 100,
+        "cls": 0.00007099999999999999,
+        "lcpMs": 3198.845,
+        "performanceScore": 89,
+        "tbtMs": 218.93099999999936
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0.00007099999999999999,
+          "lcpMs": 3198.845,
+          "performanceScore": 89,
+          "tbtMs": 218.93099999999936
+        }
+      ]
+    },
+    "http://localhost:3300/?view=design": {
+      "observation": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 2017.864,
+        "performanceScore": 96,
+        "tbtMs": 192
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 2017.864,
+          "performanceScore": 96,
+          "tbtMs": 192
+        }
+      ]
+    },
+    "http://localhost:3300/?view=editorial": {
+      "observation": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 2520.46,
+        "performanceScore": 95,
+        "tbtMs": 159.9079999999999
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 2520.46,
+          "performanceScore": 95,
+          "tbtMs": 159.9079999999999
+        }
+      ]
+    },
+    "http://localhost:3300/projects/example-project?view=brutalist": {
+      "observation": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 2919.2875000000004,
+        "performanceScore": 93,
+        "tbtMs": 145.6187500000001
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 2919.2875000000004,
+          "performanceScore": 93,
+          "tbtMs": 145.6187500000001
+        }
+      ]
+    },
+    "http://localhost:3300/projects/example-project?view=cinematic": {
+      "observation": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 2533.2645,
+        "performanceScore": 95,
+        "tbtMs": 146
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 2533.2645,
+          "performanceScore": 95,
+          "tbtMs": 146
+        }
+      ]
+    },
+    "http://localhost:3300/projects/example-project?view=classic": {
+      "observation": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 2844.5730000000003,
+        "performanceScore": 93,
+        "tbtMs": 163.5
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 2844.5730000000003,
+          "performanceScore": 93,
+          "tbtMs": 163.5
+        }
+      ]
+    },
+    "http://localhost:3300/projects/example-project?view=design": {
+      "observation": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 2063.284,
+        "performanceScore": 96,
+        "tbtMs": 190.4290000000001
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 2063.284,
+          "performanceScore": 96,
+          "tbtMs": 190.4290000000001
+        }
+      ]
+    },
+    "http://localhost:3300/projects/example-project?view=editorial": {
+      "observation": {
+        "accessibilityScore": 100,
+        "cls": 0,
+        "lcpMs": 1840.478,
+        "performanceScore": 98,
+        "tbtMs": 126.88049999999998
+      },
+      "runs": [
+        {
+          "accessibilityScore": 100,
+          "cls": 0,
+          "lcpMs": 1840.478,
+          "performanceScore": 98,
+          "tbtMs": 126.88049999999998
+        }
+      ]
+    }
+  }
+}
