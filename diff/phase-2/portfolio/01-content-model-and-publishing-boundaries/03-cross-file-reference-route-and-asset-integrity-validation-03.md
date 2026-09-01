## `test(content): Vitest 기반 콘텐츠 계약 검증 추가`

diff --git a/package-lock.json b/package-lock.json
index d40cbd7..e6a6d8d 100644
--- a/package-lock.json
+++ b/package-lock.json
@@ -16,16 +16,34 @@
       },
       "devDependencies": {
         "@tailwindcss/postcss": "^4",
+        "@testing-library/jest-dom": "^6.9.1",
+        "@testing-library/react": "^16.3.2",
         "@types/node": "^20",
         "@types/react": "^19",
         "@types/react-dom": "^19",
         "eslint": "^9",
         "eslint-config-next": "16.2.4",
+        "jsdom": "^27.0.1",
         "tailwindcss": "^4",
         "tsx": "^4.23.0",
-        "typescript": "^5"
+        "typescript": "^5",
+        "vitest": "^3.2.4"
       }
     },
+    "node_modules/@acemir/cssom": {
+      "version": "0.9.31",
+      "resolved": "https://registry.npmjs.org/@acemir/cssom/-/cssom-0.9.31.tgz",
+      "integrity": "sha512-ZnR3GSaH+/vJ0YlHau21FjfLYjMpYVIzTD8M8vIEQvIGxeOXyXdzCI140rrCY862p/C/BbzWsjc1dgnM9mkoTA==",
+      "dev": true,
+      "license": "MIT"
+    },
+    "node_modules/@adobe/css-tools": {
+      "version": "4.5.0",
+      "resolved": "https://registry.npmjs.org/@adobe/css-tools/-/css-tools-4.5.0.tgz",
+      "integrity": "sha512-6OzddxPio9UiWTCemp4N8cYLV2ZN1ncRnV1cVGtve7dhPOtRkleRyx32GQCYSwDYgaHU3USMm84tNsvKzRCa1Q==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/@alloc/quick-lru": {
       "version": "5.2.0",
       "resolved": "https://registry.npmjs.org/@alloc/quick-lru/-/quick-lru-5.2.0.tgz",
@@ -39,6 +57,61 @@
         "url": "https://github.com/sponsors/sindresorhus"
       }
     },
+    "node_modules/@asamuzakjp/css-color": {
+      "version": "4.1.2",
+      "resolved": "https://registry.npmjs.org/@asamuzakjp/css-color/-/css-color-4.1.2.tgz",
+      "integrity": "sha512-NfBUvBaYgKIuq6E/RBLY1m0IohzNHAYyaJGuTK79Z23uNwmz2jl1mPsC5ZxCCxylinKhT1Amn5oNTlx1wN8cQg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@csstools/css-calc": "^3.0.0",
+        "@csstools/css-color-parser": "^4.0.1",
+        "@csstools/css-parser-algorithms": "^4.0.0",
+        "@csstools/css-tokenizer": "^4.0.0",
+        "lru-cache": "^11.2.5"
+      }
+    },
+    "node_modules/@asamuzakjp/css-color/node_modules/lru-cache": {
+      "version": "11.5.2",
+      "resolved": "https://registry.npmjs.org/lru-cache/-/lru-cache-11.5.2.tgz",
+      "integrity": "sha512-4pfM1Ff0x50o0tQwb5ucw/RzNyD0/YJME6IVcStalZuMWxdt3sR3huStTtxz4PUmvZfRguvDejasvQ2kifR11g==",
+      "dev": true,
+      "license": "BlueOak-1.0.0",
+      "engines": {
+        "node": "20 || >=22"
+      }
+    },
+    "node_modules/@asamuzakjp/dom-selector": {
+      "version": "6.8.1",
+      "resolved": "https://registry.npmjs.org/@asamuzakjp/dom-selector/-/dom-selector-6.8.1.tgz",
+      "integrity": "sha512-MvRz1nCqW0fsy8Qz4dnLIvhOlMzqDVBabZx6lH+YywFDdjXhMY37SmpV1XFX3JzG5GWHn63j6HX6QPr3lZXHvQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@asamuzakjp/nwsapi": "^2.3.9",
+        "bidi-js": "^1.0.3",
+        "css-tree": "^3.1.0",
+        "is-potential-custom-element-name": "^1.0.1",
+        "lru-cache": "^11.2.6"
+      }
+    },
+    "node_modules/@asamuzakjp/dom-selector/node_modules/lru-cache": {
+      "version": "11.5.2",
+      "resolved": "https://registry.npmjs.org/lru-cache/-/lru-cache-11.5.2.tgz",
+      "integrity": "sha512-4pfM1Ff0x50o0tQwb5ucw/RzNyD0/YJME6IVcStalZuMWxdt3sR3huStTtxz4PUmvZfRguvDejasvQ2kifR11g==",
+      "dev": true,
+      "license": "BlueOak-1.0.0",
+      "engines": {
+        "node": "20 || >=22"
+      }
+    },
+    "node_modules/@asamuzakjp/nwsapi": {
+      "version": "2.3.9",
+      "resolved": "https://registry.npmjs.org/@asamuzakjp/nwsapi/-/nwsapi-2.3.9.tgz",
+      "integrity": "sha512-n8GuYSrI9bF7FFZ/SjhwevlHc8xaVlb/7HmHelnc/PZXBD2ZR49NnN9sMMuDdEGPeeRQ5d0hqlSlEpgCX3Wl0Q==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/@babel/code-frame": {
       "version": "7.29.0",
       "resolved": "https://registry.npmjs.org/@babel/code-frame/-/code-frame-7.29.0.tgz",
@@ -231,6 +304,16 @@
         "node": ">=6.0.0"
       }
     },
+    "node_modules/@babel/runtime": {
+      "version": "7.29.7",
+      "resolved": "https://registry.npmjs.org/@babel/runtime/-/runtime-7.29.7.tgz",
+      "integrity": "sha512-Nq8OhGWiZIZGV6hLHoyAKLLcJihP/xFeBMGJoUrxTX2psI8dCifzLhZISFb+VWS3wFMRDmCGw5R+dOySCqPLhw==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=6.9.0"
+      }
+    },
     "node_modules/@babel/template": {
       "version": "7.28.6",
       "resolved": "https://registry.npmjs.org/@babel/template/-/template-7.28.6.tgz",
@@ -279,6 +362,146 @@
         "node": ">=6.9.0"
       }
     },
+    "node_modules/@csstools/color-helpers": {
+      "version": "6.1.0",
+      "resolved": "https://registry.npmjs.org/@csstools/color-helpers/-/color-helpers-6.1.0.tgz",
+      "integrity": "sha512-064IFJdjTfUqnjpCVpMOdbr8FLQBhinbZj6yRv2An2E41O/pLEXqfFRWqGq/SxlE5PEUYTlvWsG2r8MswAVvkg==",
+      "dev": true,
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/csstools"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/csstools"
+        }
+      ],
+      "license": "MIT-0",
+      "engines": {
+        "node": ">=20.19.0"
+      }
+    },
+    "node_modules/@csstools/css-calc": {
+      "version": "3.3.0",
+      "resolved": "https://registry.npmjs.org/@csstools/css-calc/-/css-calc-3.3.0.tgz",
+      "integrity": "sha512-c5ihYsPkdG6JCkU2zTMm4+k6r7RXuGxtWYhu5DHMIiF1FHzrfmHL5so11AoFpUv/tu61xfcmT4AmKoFfMPoqdQ==",
+      "dev": true,
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/csstools"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/csstools"
+        }
+      ],
+      "license": "MIT",
+      "engines": {
+        "node": ">=20.19.0"
+      },
+      "peerDependencies": {
+        "@csstools/css-parser-algorithms": "^4.0.0",
+        "@csstools/css-tokenizer": "^4.0.0"
+      }
+    },
+    "node_modules/@csstools/css-color-parser": {
+      "version": "4.1.10",
+      "resolved": "https://registry.npmjs.org/@csstools/css-color-parser/-/css-color-parser-4.1.10.tgz",
+      "integrity": "sha512-UZhQLIUyJaaMepqehrCODwCg2KW25vFvLWBmqYFaPclYvvxzj/sG8LBOhBFCp11i9uE7t1EyS+RAoV9tztPFyw==",
+      "dev": true,
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/csstools"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/csstools"
+        }
+      ],
+      "license": "MIT",
+      "dependencies": {
+        "@csstools/color-helpers": "^6.1.0",
+        "@csstools/css-calc": "^3.3.0"
+      },
+      "engines": {
+        "node": ">=20.19.0"
+      },
+      "peerDependencies": {
+        "@csstools/css-parser-algorithms": "^4.0.0",
+        "@csstools/css-tokenizer": "^4.0.0"
+      }
+    },
+    "node_modules/@csstools/css-parser-algorithms": {
+      "version": "4.0.0",
+      "resolved": "https://registry.npmjs.org/@csstools/css-parser-algorithms/-/css-parser-algorithms-4.0.0.tgz",
+      "integrity": "sha512-+B87qS7fIG3L5h3qwJ/IFbjoVoOe/bpOdh9hAjXbvx0o8ImEmUsGXN0inFOnk2ChCFgqkkGFQ+TpM5rbhkKe4w==",
+      "dev": true,
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/csstools"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/csstools"
+        }
+      ],
+      "license": "MIT",
+      "engines": {
+        "node": ">=20.19.0"
+      },
+      "peerDependencies": {
+        "@csstools/css-tokenizer": "^4.0.0"
+      }
+    },
+    "node_modules/@csstools/css-syntax-patches-for-csstree": {
+      "version": "1.1.7",
+      "resolved": "https://registry.npmjs.org/@csstools/css-syntax-patches-for-csstree/-/css-syntax-patches-for-csstree-1.1.7.tgz",
+      "integrity": "sha512-fQ+05118eQS1cofO3aJpB5efgpBZMvIzwr/sbC8kDLVA5XLG8q1kJV5yzrUAI1f7lvhPnm8fgIjzFB8/O/5Dig==",
+      "dev": true,
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/csstools"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/csstools"
+        }
+      ],
+      "license": "MIT-0",
+      "peerDependencies": {
+        "css-tree": "^3.2.1"
+      },
+      "peerDependenciesMeta": {
+        "css-tree": {
+          "optional": true
+        }
+      }
+    },
+    "node_modules/@csstools/css-tokenizer": {
+      "version": "4.0.0",
+      "resolved": "https://registry.npmjs.org/@csstools/css-tokenizer/-/css-tokenizer-4.0.0.tgz",
+      "integrity": "sha512-QxULHAm7cNu72w97JUNCBFODFaXpbDg+dP8b/oWFAZ2MTRppA3U00Y2L1HqaS4J6yBqxwa/Y3nMBaxVKbB/NsA==",
+      "dev": true,
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/csstools"
+        },
+        {
+          "type": "opencollective",
+          "url": "https://opencollective.com/csstools"
+        }
+      ],
+      "license": "MIT",
+      "engines": {
+        "node": ">=20.19.0"
+      }
+    },
     "node_modules/@emnapi/core": {
       "version": "1.10.0",
       "resolved": "https://registry.npmjs.org/@emnapi/core/-/core-1.10.0.tgz",
@@ -898,6 +1121,24 @@
         "node": "^18.18.0 || ^20.9.0 || >=21.1.0"
       }
     },
+    "node_modules/@exodus/bytes": {
+      "version": "1.15.1",
+      "resolved": "https://registry.npmjs.org/@exodus/bytes/-/bytes-1.15.1.tgz",
+      "integrity": "sha512-S6mL0yNB/Abt9Ei4tq8gDhcczc4S3+vQ4ra7vxnAf+YHC02srtqxKKZghx2Dq6p0e66THKwR6r8N6P95wEty7Q==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": "^20.19.0 || ^22.12.0 || >=24.0.0"
+      },
+      "peerDependencies": {
+        "@noble/hashes": "^1.8.0 || ^2.0.0"
+      },
+      "peerDependenciesMeta": {
+        "@noble/hashes": {
+          "optional": true
+        }
+      }
+    },
     "node_modules/@humanfs/core": {
       "version": "0.19.2",
       "resolved": "https://registry.npmjs.org/@humanfs/core/-/core-0.19.2.tgz",
@@ -1480,6 +1721,26 @@
         "@jridgewell/sourcemap-codec": "^1.4.14"
       }
     },
+    "node_modules/@napi-rs/lzma-linux-x64-gnu": {
+      "version": "1.5.1",
+      "resolved": "https://registry.npmjs.org/@napi-rs/lzma-linux-x64-gnu/-/lzma-linux-x64-gnu-1.5.1.tgz",
+      "integrity": "sha512-oTXEIha4SsuXdTA4Iyskj0kpdx2yVXdhd75c2v3xGrHFfVMsbhTPZU/nMPL4sWKo4pBHm3aucLaqGlF696dTyQ==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "libc": [
+        "glibc"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": "^22.20 || ^24.12 || >=25"
+      }
+    },
     "node_modules/@napi-rs/wasm-runtime": {
       "version": "0.2.12",
       "resolved": "https://registry.npmjs.org/@napi-rs/wasm-runtime/-/wasm-runtime-0.2.12.tgz",
@@ -1685,66 +1946,24 @@
         "node": ">=12.4.0"
       }
     },
-    "node_modules/@rtsao/scc": {
-      "version": "1.1.0",
-      "resolved": "https://registry.npmjs.org/@rtsao/scc/-/scc-1.1.0.tgz",
-      "integrity": "sha512-zt6OdqaDoOnJ1ZYsCYGt9YmWzDXl4vQdKTyJev62gFhRGKdx7mcT54V9KIjg+d2wi9EXsPvAPKe7i7WjfVWB8g==",
-      "dev": true,
-      "license": "MIT"
-    },
-    "node_modules/@swc/helpers": {
-      "version": "0.5.15",
-      "resolved": "https://registry.npmjs.org/@swc/helpers/-/helpers-0.5.15.tgz",
-      "integrity": "sha512-JQ5TuMi45Owi4/BIMAJBoSQoOJu12oOk/gADqlcUL9JEdHB8vyjUSsxqeNXnmXHjYKMi2WcYtezGEEhqUI/E2g==",
-      "license": "Apache-2.0",
-      "dependencies": {
-        "tslib": "^2.8.0"
-      }
-    },
-    "node_modules/@tailwindcss/node": {
-      "version": "4.2.4",
-      "resolved": "https://registry.npmjs.org/@tailwindcss/node/-/node-4.2.4.tgz",
-      "integrity": "sha512-Ai7+yQPxz3ddrDQzFfBKdHEVBg0w3Zl83jnjuwxnZOsnH9pGn93QHQtpU0p/8rYWxvbFZHneni6p1BSLK4DkGA==",
-      "dev": true,
-      "license": "MIT",
-      "dependencies": {
-        "@jridgewell/remapping": "^2.3.5",
-        "enhanced-resolve": "^5.19.0",
-        "jiti": "^2.6.1",
-        "lightningcss": "1.32.0",
-        "magic-string": "^0.30.21",
-        "source-map-js": "^1.2.1",
-        "tailwindcss": "4.2.4"
-      }
-    },
-    "node_modules/@tailwindcss/oxide": {
-      "version": "4.2.4",
-      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide/-/oxide-4.2.4.tgz",
-      "integrity": "sha512-9El/iI069DKDSXwTvB9J4BwdO5JhRrOweGaK25taBAvBXyXqJAX+Jqdvs8r8gKpsI/1m0LeJLyQYTf/WLrBT1Q==",
+    "node_modules/@rollup/rollup-android-arm-eabi": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-android-arm-eabi/-/rollup-android-arm-eabi-4.62.4.tgz",
+      "integrity": "sha512-RrPokAb7dmbxFoeO3TloqHyOjgye8RkBhSqmp4aJMIex4c9r46ZstPnleDQOq1t46VOVjwIuwNogIqbodV1Vvg==",
+      "cpu": [
+        "arm"
+      ],
       "dev": true,
       "license": "MIT",
-      "engines": {
-        "node": ">= 20"
-      },
-      "optionalDependencies": {
-        "@tailwindcss/oxide-android-arm64": "4.2.4",
-        "@tailwindcss/oxide-darwin-arm64": "4.2.4",
-        "@tailwindcss/oxide-darwin-x64": "4.2.4",
-        "@tailwindcss/oxide-freebsd-x64": "4.2.4",
-        "@tailwindcss/oxide-linux-arm-gnueabihf": "4.2.4",
-        "@tailwindcss/oxide-linux-arm64-gnu": "4.2.4",
-        "@tailwindcss/oxide-linux-arm64-musl": "4.2.4",
-        "@tailwindcss/oxide-linux-x64-gnu": "4.2.4",
-        "@tailwindcss/oxide-linux-x64-musl": "4.2.4",
-        "@tailwindcss/oxide-wasm32-wasi": "4.2.4",
-        "@tailwindcss/oxide-win32-arm64-msvc": "4.2.4",
-        "@tailwindcss/oxide-win32-x64-msvc": "4.2.4"
-      }
+      "optional": true,
+      "os": [
+        "android"
+      ]
     },
-    "node_modules/@tailwindcss/oxide-android-arm64": {
-      "version": "4.2.4",
-      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-android-arm64/-/oxide-android-arm64-4.2.4.tgz",
-      "integrity": "sha512-e7MOr1SAn9U8KlZzPi1ZXGZHeC5anY36qjNwmZv9pOJ8E4Q6jmD1vyEHkQFmNOIN7twGPEMXRHmitN4zCMN03g==",
+    "node_modules/@rollup/rollup-android-arm64": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-android-arm64/-/rollup-android-arm64-4.62.4.tgz",
+      "integrity": "sha512-JKuJc+pnpks2pjy7L/N3v/cAkZxYlnmuZoD840ldbMI5KDbC4iO9NKwPKYdjYFCMAIIlBzYSFHxIJVYzRo2/8A==",
       "cpu": [
         "arm64"
       ],
@@ -1753,15 +1972,12 @@
       "optional": true,
       "os": [
         "android"
-      ],
-      "engines": {
-        "node": ">= 20"
-      }
+      ]
     },
-    "node_modules/@tailwindcss/oxide-darwin-arm64": {
-      "version": "4.2.4",
-      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-darwin-arm64/-/oxide-darwin-arm64-4.2.4.tgz",
-      "integrity": "sha512-tSC/Kbqpz/5/o/C2sG7QvOxAKqyd10bq+ypZNf+9Fi2TvbVbv1zNpcEptcsU7DPROaSbVgUXmrzKhurFvo5eDg==",
+    "node_modules/@rollup/rollup-darwin-arm64": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-darwin-arm64/-/rollup-darwin-arm64-4.62.4.tgz",
+      "integrity": "sha512-krw5uS2STmvJ02x0uTXHbqQNuz+9eZ1iw+qXk9dmW2gvV4jV7O2hEoOnuhFrpOPiel1mBFtqbxYZZtC46hXLOw==",
       "cpu": [
         "arm64"
       ],
@@ -1770,15 +1986,12 @@
       "optional": true,
       "os": [
         "darwin"
-      ],
-      "engines": {
-        "node": ">= 20"
-      }
+      ]
     },
-    "node_modules/@tailwindcss/oxide-darwin-x64": {
-      "version": "4.2.4",
-      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-darwin-x64/-/oxide-darwin-x64-4.2.4.tgz",
-      "integrity": "sha512-yPyUXn3yO/ufR6+Kzv0t4fCg2qNr90jxXc5QqBpjlPNd0NqyDXcmQb/6weunH/MEDXW5dhyEi+agTDiqa3WsGg==",
+    "node_modules/@rollup/rollup-darwin-x64": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-darwin-x64/-/rollup-darwin-x64-4.62.4.tgz",
+      "integrity": "sha512-wsTxtgApb4PrOsNJIm0FZ1h3WvCC+k9uxLJ4ad75hgoS4NiRes2SoJFlDAyMwiUY8IssDqGcHbXuN0sx1tfF1A==",
       "cpu": [
         "x64"
       ],
@@ -1787,41 +2000,478 @@
       "optional": true,
       "os": [
         "darwin"
-      ],
-      "engines": {
-        "node": ">= 20"
-      }
+      ]
     },
-    "node_modules/@tailwindcss/oxide-freebsd-x64": {
-      "version": "4.2.4",
-      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-freebsd-x64/-/oxide-freebsd-x64-4.2.4.tgz",
-      "integrity": "sha512-BoMIB4vMQtZsXdGLVc2z+P9DbETkiopogfWZKbWwM8b/1Vinbs4YcUwo+kM/KeLkX3Ygrf4/PsRndKaYhS8Eiw==",
+    "node_modules/@rollup/rollup-freebsd-arm64": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-freebsd-arm64/-/rollup-freebsd-arm64-4.62.4.tgz",
+      "integrity": "sha512-GUOnQlyZe3yAXhWOtOMsn5Qkrv5E5mZXa0thbARWi5Ei2szlVXJFQhddZ4HbAzh8q92w5twp+CQvs/eFanz9YQ==",
       "cpu": [
-        "x64"
+        "arm64"
       ],
       "dev": true,
       "license": "MIT",
       "optional": true,
       "os": [
         "freebsd"
-      ],
-      "engines": {
-        "node": ">= 20"
-      }
+      ]
     },
-    "node_modules/@tailwindcss/oxide-linux-arm-gnueabihf": {
-      "version": "4.2.4",
-      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-linux-arm-gnueabihf/-/oxide-linux-arm-gnueabihf-4.2.4.tgz",
-      "integrity": "sha512-7pIHBLTHYRAlS7V22JNuTh33yLH4VElwKtB3bwchK/UaKUPpQ0lPQiOWcbm4V3WP2I6fNIJ23vABIvoy2izdwA==",
+    "node_modules/@rollup/rollup-freebsd-x64": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-freebsd-x64/-/rollup-freebsd-x64-4.62.4.tgz",
+      "integrity": "sha512-/Y7f3QuxjzPKsjA/rfEDa3+0vXqyjmJ50Ln8dPpCmWkKTrUoWHG1cWhTqaAMLob2m2nESWuC7yGrREz019Ztqg==",
       "cpu": [
-        "arm"
+        "x64"
       ],
       "dev": true,
       "license": "MIT",
       "optional": true,
       "os": [
-        "linux"
-      ],
+        "freebsd"
+      ]
+    },
+    "node_modules/@rollup/rollup-linux-arm-gnueabihf": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm-gnueabihf/-/rollup-linux-arm-gnueabihf-4.62.4.tgz",
+      "integrity": "sha512-81wiiX3v7aqy+T+bT61TJ78yJjRquqFFTTbAPt08imfQQzkPIW8t6aJbkTagtCCrXMNc9D66+geqlK7ydLPNqA==",
+      "cpu": [
+        "arm"
+      ],
+      "dev": true,
+      "libc": [
+        "glibc"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ]
+    },
+    "node_modules/@rollup/rollup-linux-arm-musleabihf": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm-musleabihf/-/rollup-linux-arm-musleabihf-4.62.4.tgz",
+      "integrity": "sha512-9kmDIvNZqdoHOBZgNtpTBeLWYO/LVipM3H/j62P8848/l/VPEQL6N3uxU9pvP1oZAsXyC2MEnFP3ovRjo7WYNQ==",
+      "cpu": [
+        "arm"
+      ],
+      "dev": true,
+      "libc": [
+        "musl"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ]
+    },
+    "node_modules/@rollup/rollup-linux-arm64-gnu": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm64-gnu/-/rollup-linux-arm64-gnu-4.62.4.tgz",
+      "integrity": "sha512-CcnXHWnXg69g+DX5VWL3FHts3qMRN2uVEHX+BZvGLdd07/gXkn3ePjYtO1LDJvxkGKVHMclKBRa1QUTH+6toYQ==",
+      "cpu": [
+        "arm64"
+      ],
+      "dev": true,
+      "libc": [
+        "glibc"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ]
+    },
+    "node_modules/@rollup/rollup-linux-arm64-musl": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm64-musl/-/rollup-linux-arm64-musl-4.62.4.tgz",
+      "integrity": "sha512-iFOibiHnTRuhrWLlRsOQFdZJJIa7S8OwkneJr4ocALP16u5yk6lWLINFwhHaEqBFMsKDUZofLkGos7+CPzGB3g==",
+      "cpu": [
+        "arm64"
+      ],
+      "dev": true,
+      "libc": [
+        "musl"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ]
+    },
+    "node_modules/@rollup/rollup-linux-loong64-gnu": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-loong64-gnu/-/rollup-linux-loong64-gnu-4.62.4.tgz",
+      "integrity": "sha512-XnWYMI7euHlb5a871xPja+Gm7DRCFU+FGRrtS2sMq9N8FvqtpagUy6gD4YOemC5MRk9xbh8+jYMEJbigFQwsgA==",
+      "cpu": [
+        "loong64"
+      ],
+      "dev": true,
+      "libc": [
+        "glibc"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ]
+    },
+    "node_modules/@rollup/rollup-linux-loong64-musl": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-loong64-musl/-/rollup-linux-loong64-musl-4.62.4.tgz",
+      "integrity": "sha512-qGDAlO0U8xedCcsdRm9oaoQY8DAx/QT7uIxJWhCdx0ceIWX783UC9QSYkdpzAe29wNiVfp24+bZdQmn49o45SQ==",
+      "cpu": [
+        "loong64"
+      ],
+      "dev": true,
+      "libc": [
+        "musl"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ]
+    },
+    "node_modules/@rollup/rollup-linux-ppc64-gnu": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-ppc64-gnu/-/rollup-linux-ppc64-gnu-4.62.4.tgz",
+      "integrity": "sha512-ru4H6ezD7ysA5EiEK6qkkaEb4modH8CTej6kUy/gQi20u3kB3G7Zn8snXXkeJSCOFKG/rbPPtM/+9Wgas1961w==",
+      "cpu": [
+        "ppc64"
+      ],
+      "dev": true,
+      "libc": [
+        "glibc"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ]
+    },
+    "node_modules/@rollup/rollup-linux-ppc64-musl": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-ppc64-musl/-/rollup-linux-ppc64-musl-4.62.4.tgz",
+      "integrity": "sha512-2W4MO5WQVJnbJaZdvDb9rhBDuFU1nKIepPFpJUBsTh2k1YY2g+ODViaWuyOAjQ5cOP7NvrvLzt3wvHOoiAvc7w==",
+      "cpu": [
+        "ppc64"
+      ],
+      "dev": true,
+      "libc": [
+        "musl"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ]
+    },
+    "node_modules/@rollup/rollup-linux-riscv64-gnu": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-riscv64-gnu/-/rollup-linux-riscv64-gnu-4.62.4.tgz",
+      "integrity": "sha512-+fxjfuoAmVMCYV5QyjoIpu0cp5DOiOTeqYFk1AVaxGr+/ravWLX89XfQmptsoWcaVy/TGf2hexzbUOrCQIL1CQ==",
+      "cpu": [
+        "riscv64"
+      ],
+      "dev": true,
+      "libc": [
+        "glibc"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ]
+    },
+    "node_modules/@rollup/rollup-linux-riscv64-musl": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-riscv64-musl/-/rollup-linux-riscv64-musl-4.62.4.tgz",
+      "integrity": "sha512-jTn8JfHGL4djjFxPuM06LmNUJDsst2jeVlsd9OmIH6zc5sC9K6rIuO4YajXatLUpBmBKl6b35ro1QZocLi+tcA==",
+      "cpu": [
+        "riscv64"
+      ],
+      "dev": true,
+      "libc": [
+        "musl"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ]
+    },
+    "node_modules/@rollup/rollup-linux-s390x-gnu": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-s390x-gnu/-/rollup-linux-s390x-gnu-4.62.4.tgz",
+      "integrity": "sha512-oCJCJL4pXsoDcP2QZ+JVlPTIRc6266zsIaeJJsWImmF7HO0W8nb6HuSgZlMWxJwaPf8ehbSw8yo0EUw925hKsA==",
+      "cpu": [
+        "s390x"
+      ],
+      "dev": true,
+      "libc": [
+        "glibc"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ]
+    },
+    "node_modules/@rollup/rollup-linux-x64-gnu": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-x64-gnu/-/rollup-linux-x64-gnu-4.62.4.tgz",
+      "integrity": "sha512-W69hukhZ3KKNRCaMIEzKvcFye42hh0FE1+YoYaf5+Ikacuftoco6yO/xouz0hc5d5W/s3yBro5jRiuEE/Q5vUw==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "libc": [
+        "glibc"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ]
+    },
+    "node_modules/@rollup/rollup-linux-x64-musl": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-x64-musl/-/rollup-linux-x64-musl-4.62.4.tgz",
+      "integrity": "sha512-qiXbGG2jkjXhzXpsFZSR2Xpb8DN/UaxYsbb/STbuR/6fpaDgRmmaq1B/LmtF2wQFOFOSsK2jdE0RZ3a0zHn4QA==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "libc": [
+        "musl"
+      ],
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ]
+    },
+    "node_modules/@rollup/rollup-openbsd-x64": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-openbsd-x64/-/rollup-openbsd-x64-4.62.4.tgz",
+      "integrity": "sha512-nWeM//hxv8mIo6jD7Hu4o48DVmV9pbV6gsKaWU+4NFyqHoPKwrkRiZGLKUhOBk8qNmDmpwFtPKg80Bo/Tn4xiQ==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "openbsd"
+      ]
+    },
+    "node_modules/@rollup/rollup-openharmony-arm64": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-openharmony-arm64/-/rollup-openharmony-arm64-4.62.4.tgz",
+      "integrity": "sha512-s62SQ/vgsRSvMwDkOEfTqfgASF0f26ZNaQuTA6Aok5lrikf89yI2W0gFHvZb2Jpgc6N8JnOKZgCK2iciO3CsxQ==",
+      "cpu": [
+        "arm64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "openharmony"
+      ]
+    },
+    "node_modules/@rollup/rollup-win32-arm64-msvc": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-arm64-msvc/-/rollup-win32-arm64-msvc-4.62.4.tgz",
+      "integrity": "sha512-J6wGf8TVGbXJq+HH+ttTvrcfNKPbuZecV6KT1B8I18BC5IURUh5kl4Yl5OEP5eFIUoI5BWxCsyYMhFsDx8kekw==",
+      "cpu": [
+        "arm64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "win32"
+      ]
+    },
+    "node_modules/@rollup/rollup-win32-ia32-msvc": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-ia32-msvc/-/rollup-win32-ia32-msvc-4.62.4.tgz",
+      "integrity": "sha512-zmfrQd/0wu6oJs8Vq8KwY/YtsKSsLtKe/HwAP4Wqy8LhWjeT55fHRAkOhYQ12wI3ayS4Tt12d5CDRD7N96SAYQ==",
+      "cpu": [
+        "ia32"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "win32"
+      ]
+    },
+    "node_modules/@rollup/rollup-win32-x64-gnu": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-x64-gnu/-/rollup-win32-x64-gnu-4.62.4.tgz",
+      "integrity": "sha512-qPzHqdj9rfUD+w79dtE07zi/kFwKyCJqplp5K5ygeLTp7jLpAoc16OAH39HSmRC9UpozaecsleI8uAdEj6v2yw==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "win32"
+      ]
+    },
+    "node_modules/@rollup/rollup-win32-x64-msvc": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-x64-msvc/-/rollup-win32-x64-msvc-4.62.4.tgz",
+      "integrity": "sha512-zD6NdeWEByGE9QF9vCrlJ5YQB4oq9q91kPZS37Jwj5hOkvR1lTBSpsKhKDw4IJtbQ35LsTS1HD9DZYGKIshU1Q==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "win32"
+      ]
+    },
+    "node_modules/@rtsao/scc": {
+      "version": "1.1.0",
+      "resolved": "https://registry.npmjs.org/@rtsao/scc/-/scc-1.1.0.tgz",
+      "integrity": "sha512-zt6OdqaDoOnJ1ZYsCYGt9YmWzDXl4vQdKTyJev62gFhRGKdx7mcT54V9KIjg+d2wi9EXsPvAPKe7i7WjfVWB8g==",
+      "dev": true,
+      "license": "MIT"
+    },
+    "node_modules/@swc/helpers": {
+      "version": "0.5.15",
+      "resolved": "https://registry.npmjs.org/@swc/helpers/-/helpers-0.5.15.tgz",
+      "integrity": "sha512-JQ5TuMi45Owi4/BIMAJBoSQoOJu12oOk/gADqlcUL9JEdHB8vyjUSsxqeNXnmXHjYKMi2WcYtezGEEhqUI/E2g==",
+      "license": "Apache-2.0",
+      "dependencies": {
+        "tslib": "^2.8.0"
+      }
+    },
+    "node_modules/@tailwindcss/node": {
+      "version": "4.2.4",
+      "resolved": "https://registry.npmjs.org/@tailwindcss/node/-/node-4.2.4.tgz",
+      "integrity": "sha512-Ai7+yQPxz3ddrDQzFfBKdHEVBg0w3Zl83jnjuwxnZOsnH9pGn93QHQtpU0p/8rYWxvbFZHneni6p1BSLK4DkGA==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@jridgewell/remapping": "^2.3.5",
+        "enhanced-resolve": "^5.19.0",
+        "jiti": "^2.6.1",
+        "lightningcss": "1.32.0",
+        "magic-string": "^0.30.21",
+        "source-map-js": "^1.2.1",
+        "tailwindcss": "4.2.4"
+      }
+    },
+    "node_modules/@tailwindcss/oxide": {
+      "version": "4.2.4",
+      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide/-/oxide-4.2.4.tgz",
+      "integrity": "sha512-9El/iI069DKDSXwTvB9J4BwdO5JhRrOweGaK25taBAvBXyXqJAX+Jqdvs8r8gKpsI/1m0LeJLyQYTf/WLrBT1Q==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">= 20"
+      },
+      "optionalDependencies": {
+        "@tailwindcss/oxide-android-arm64": "4.2.4",
+        "@tailwindcss/oxide-darwin-arm64": "4.2.4",
+        "@tailwindcss/oxide-darwin-x64": "4.2.4",
+        "@tailwindcss/oxide-freebsd-x64": "4.2.4",
+        "@tailwindcss/oxide-linux-arm-gnueabihf": "4.2.4",
+        "@tailwindcss/oxide-linux-arm64-gnu": "4.2.4",
+        "@tailwindcss/oxide-linux-arm64-musl": "4.2.4",
+        "@tailwindcss/oxide-linux-x64-gnu": "4.2.4",
+        "@tailwindcss/oxide-linux-x64-musl": "4.2.4",
+        "@tailwindcss/oxide-wasm32-wasi": "4.2.4",
+        "@tailwindcss/oxide-win32-arm64-msvc": "4.2.4",
+        "@tailwindcss/oxide-win32-x64-msvc": "4.2.4"
+      }
+    },
+    "node_modules/@tailwindcss/oxide-android-arm64": {
+      "version": "4.2.4",
+      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-android-arm64/-/oxide-android-arm64-4.2.4.tgz",
+      "integrity": "sha512-e7MOr1SAn9U8KlZzPi1ZXGZHeC5anY36qjNwmZv9pOJ8E4Q6jmD1vyEHkQFmNOIN7twGPEMXRHmitN4zCMN03g==",
+      "cpu": [
+        "arm64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "android"
+      ],
+      "engines": {
+        "node": ">= 20"
+      }
+    },
+    "node_modules/@tailwindcss/oxide-darwin-arm64": {
+      "version": "4.2.4",
+      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-darwin-arm64/-/oxide-darwin-arm64-4.2.4.tgz",
+      "integrity": "sha512-tSC/Kbqpz/5/o/C2sG7QvOxAKqyd10bq+ypZNf+9Fi2TvbVbv1zNpcEptcsU7DPROaSbVgUXmrzKhurFvo5eDg==",
+      "cpu": [
+        "arm64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "darwin"
+      ],
+      "engines": {
+        "node": ">= 20"
+      }
+    },
+    "node_modules/@tailwindcss/oxide-darwin-x64": {
+      "version": "4.2.4",
+      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-darwin-x64/-/oxide-darwin-x64-4.2.4.tgz",
+      "integrity": "sha512-yPyUXn3yO/ufR6+Kzv0t4fCg2qNr90jxXc5QqBpjlPNd0NqyDXcmQb/6weunH/MEDXW5dhyEi+agTDiqa3WsGg==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "darwin"
+      ],
+      "engines": {
+        "node": ">= 20"
+      }
+    },
+    "node_modules/@tailwindcss/oxide-freebsd-x64": {
+      "version": "4.2.4",
+      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-freebsd-x64/-/oxide-freebsd-x64-4.2.4.tgz",
+      "integrity": "sha512-BoMIB4vMQtZsXdGLVc2z+P9DbETkiopogfWZKbWwM8b/1Vinbs4YcUwo+kM/KeLkX3Ygrf4/PsRndKaYhS8Eiw==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "freebsd"
+      ],
+      "engines": {
+        "node": ">= 20"
+      }
+    },
+    "node_modules/@tailwindcss/oxide-linux-arm-gnueabihf": {
+      "version": "4.2.4",
+      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-linux-arm-gnueabihf/-/oxide-linux-arm-gnueabihf-4.2.4.tgz",
+      "integrity": "sha512-7pIHBLTHYRAlS7V22JNuTh33yLH4VElwKtB3bwchK/UaKUPpQ0lPQiOWcbm4V3WP2I6fNIJ23vABIvoy2izdwA==",
+      "cpu": [
+        "arm"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
       "engines": {
         "node": ">= 20"
       }
@@ -1972,6 +2622,93 @@
         "tailwindcss": "4.2.4"
       }
     },
+    "node_modules/@testing-library/dom": {
+      "version": "10.4.1",
+      "resolved": "https://registry.npmjs.org/@testing-library/dom/-/dom-10.4.1.tgz",
+      "integrity": "sha512-o4PXJQidqJl82ckFaXUeoAW+XysPLauYI43Abki5hABd853iMhitooc6znOnczgbTYmEP6U6/y1ZyKAIsvMKGg==",
+      "dev": true,
+      "license": "MIT",
+      "peer": true,
+      "dependencies": {
+        "@babel/code-frame": "^7.10.4",
+        "@babel/runtime": "^7.12.5",
+        "@types/aria-query": "^5.0.1",
+        "aria-query": "5.3.0",
+        "dom-accessibility-api": "^0.5.9",
+        "lz-string": "^1.5.0",
+        "picocolors": "1.1.1",
+        "pretty-format": "^27.0.2"
+      },
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/@testing-library/dom/node_modules/aria-query": {
+      "version": "5.3.0",
+      "resolved": "https://registry.npmjs.org/aria-query/-/aria-query-5.3.0.tgz",
+      "integrity": "sha512-b0P0sZPKtyu8HkeRAfCq0IfURZK+SuwMjY1UXGBU27wpAiTwQAIlq56IbIO+ytk/JjS1fMR14ee5WBBfKi5J6A==",
+      "dev": true,
+      "license": "Apache-2.0",
+      "peer": true,
+      "dependencies": {
+        "dequal": "^2.0.3"
+      }
+    },
+    "node_modules/@testing-library/jest-dom": {
+      "version": "6.9.1",
+      "resolved": "https://registry.npmjs.org/@testing-library/jest-dom/-/jest-dom-6.9.1.tgz",
+      "integrity": "sha512-zIcONa+hVtVSSep9UT3jZ5rizo2BsxgyDYU7WFD5eICBE7no3881HGeb/QkGfsJs6JTkY1aQhT7rIPC7e+0nnA==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@adobe/css-tools": "^4.4.0",
+        "aria-query": "^5.0.0",
+        "css.escape": "^1.5.1",
+        "dom-accessibility-api": "^0.6.3",
+        "picocolors": "^1.1.1",
+        "redent": "^3.0.0"
+      },
+      "engines": {
+        "node": ">=14",
+        "npm": ">=6",
+        "yarn": ">=1"
+      }
+    },
+    "node_modules/@testing-library/jest-dom/node_modules/dom-accessibility-api": {
+      "version": "0.6.3",
+      "resolved": "https://registry.npmjs.org/dom-accessibility-api/-/dom-accessibility-api-0.6.3.tgz",
+      "integrity": "sha512-7ZgogeTnjuHbo+ct10G9Ffp0mif17idi0IyWNVA/wcwcm7NPOD/WEHVP3n7n3MhXqxoIYm8d6MuZohYWIZ4T3w==",
+      "dev": true,
+      "license": "MIT"
+    },
+    "node_modules/@testing-library/react": {
+      "version": "16.3.2",
+      "resolved": "https://registry.npmjs.org/@testing-library/react/-/react-16.3.2.tgz",
+      "integrity": "sha512-XU5/SytQM+ykqMnAnvB2umaJNIOsLF3PVv//1Ew4CTcpz0/BRyy/af40qqrt7SjKpDdT1saBMc42CUok5gaw+g==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@babel/runtime": "^7.12.5"
+      },
+      "engines": {
+        "node": ">=18"
+      },
+      "peerDependencies": {
+        "@testing-library/dom": "^10.0.0",
+        "@types/react": "^18.0.0 || ^19.0.0",
+        "@types/react-dom": "^18.0.0 || ^19.0.0",
+        "react": "^18.0.0 || ^19.0.0",
+        "react-dom": "^18.0.0 || ^19.0.0"
+      },
+      "peerDependenciesMeta": {
+        "@types/react": {
+          "optional": true
+        },
+        "@types/react-dom": {
+          "optional": true
+        }
+      }
+    },
     "node_modules/@tybys/wasm-util": {
       "version": "0.10.2",
       "resolved": "https://registry.npmjs.org/@tybys/wasm-util/-/wasm-util-0.10.2.tgz",
@@ -1983,10 +2720,36 @@
         "tslib": "^2.4.0"
       }
     },
+    "node_modules/@types/aria-query": {
+      "version": "5.0.4",
+      "resolved": "https://registry.npmjs.org/@types/aria-query/-/aria-query-5.0.4.tgz",
+      "integrity": "sha512-rfT93uj5s0PRL7EzccGMs3brplhcrghnDoV26NqKhCAS1hVo+WdNsPvE/yb6ilfr5hi2MEk6d5EWJTKdxg8jVw==",
+      "dev": true,
+      "license": "MIT",
+      "peer": true
+    },
+    "node_modules/@types/chai": {
+      "version": "5.2.3",
+      "resolved": "https://registry.npmjs.org/@types/chai/-/chai-5.2.3.tgz",
+      "integrity": "sha512-Mw558oeA9fFbv65/y4mHtXDs9bPnFMZAL/jxdPFUpOHHIXX91mcgEHbS5Lahr+pwZFR8A7GQleRWeI6cGFC2UA==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@types/deep-eql": "*",
+        "assertion-error": "^2.0.1"
+      }
+    },
+    "node_modules/@types/deep-eql": {
+      "version": "4.0.2",
+      "resolved": "https://registry.npmjs.org/@types/deep-eql/-/deep-eql-4.0.2.tgz",
+      "integrity": "sha512-c9h9dVVMigMPc4bwTvC5dxqtqJZwQPePsWjPlpSOnojbor6pGqdk541lfA7AqFQr5pB1BRdq0juY9db81BwyFw==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/@types/estree": {
-      "version": "1.0.8",
-      "resolved": "https://registry.npmjs.org/@types/estree/-/estree-1.0.8.tgz",
-      "integrity": "sha512-dWHzHa2WqEXI/O1E9OjrocMTKJl2mSrEolh1Iomrv6U+JuNwaHXsXx9bLu5gG7BUWFIN0skIQJQ/L1rIex4X6w==",
+      "version": "1.0.9",
+      "resolved": "https://registry.npmjs.org/@types/estree/-/estree-1.0.9.tgz",
+      "integrity": "sha512-GhdPgy1el4/ImP05X05Uw4cw2/M93BCUmnEvWZNStlCzEKME4Fkk+YpoA5OiHNQmoS7Cafb8Xa3Pya8m1Qrzeg==",
       "dev": true,
       "license": "MIT"
     },
@@ -2598,6 +3361,121 @@
         "win32"
       ]
     },
+    "node_modules/@vitest/expect": {
+      "version": "3.2.7",
+      "resolved": "https://registry.npmjs.org/@vitest/expect/-/expect-3.2.7.tgz",
+      "integrity": "sha512-E8eBXaKibuvH2pSZErOjdVb5vF4PbKYcrnluBTYxEk1l/VhhwZg1kZQsdtjq+CsF5CFydf2Rdkz7jDHKSisi3w==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@types/chai": "^5.2.2",
+        "@vitest/spy": "3.2.7",
+        "@vitest/utils": "3.2.7",
+        "chai": "^5.2.0",
+        "tinyrainbow": "^2.0.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/vitest"
+      }
+    },
+    "node_modules/@vitest/mocker": {
+      "version": "3.2.7",
+      "resolved": "https://registry.npmjs.org/@vitest/mocker/-/mocker-3.2.7.tgz",
+      "integrity": "sha512-Trr0hYO9CM3Wj6ksWHRhK9IZpIY6wTMO5u/MqXurMxT57sWBaOPEtP3Oq60ihZuh5JsiagKfz95OcxdEP6dBrA==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@vitest/spy": "3.2.7",
+        "estree-walker": "^3.0.3",
+        "magic-string": "^0.30.17"
+      },
+      "funding": {
+        "url": "https://opencollective.com/vitest"
+      },
+      "peerDependencies": {
+        "msw": "^2.4.9",
+        "vite": "^5.0.0 || ^6.0.0 || ^7.0.0-0"
+      },
+      "peerDependenciesMeta": {
+        "msw": {
+          "optional": true
+        },
+        "vite": {
+          "optional": true
+        }
+      }
+    },
+    "node_modules/@vitest/pretty-format": {
+      "version": "3.2.7",
+      "resolved": "https://registry.npmjs.org/@vitest/pretty-format/-/pretty-format-3.2.7.tgz",
+      "integrity": "sha512-KUHlwqVu0sRlhCdyPdQ/wBoTfRahjUky1MubOmYw9fWfIZy1gNoHpuaaQBPAaMaVYdQYHJLurzj8ECCj5OwTqA==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "tinyrainbow": "^2.0.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/vitest"
+      }
+    },
+    "node_modules/@vitest/runner": {
+      "version": "3.2.7",
+      "resolved": "https://registry.npmjs.org/@vitest/runner/-/runner-3.2.7.tgz",
+      "integrity": "sha512-sB9y4ovltoQP+WaUPwmSxO9WIg9Ig694Di5PalVPsYHklAdE027mehpWF2SQSVq+k6sFgaivbTjTJwZLSHbedA==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@vitest/utils": "3.2.7",
+        "pathe": "^2.0.3",
+        "strip-literal": "^3.0.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/vitest"
+      }
+    },
+    "node_modules/@vitest/snapshot": {
+      "version": "3.2.7",
+      "resolved": "https://registry.npmjs.org/@vitest/snapshot/-/snapshot-3.2.7.tgz",
+      "integrity": "sha512-7C+MwShwtBSI5Buwoyg3s/iY1eHL9PKAf+O1wVh/TdnjXUtkoL/9YQtre90i4MtNXM6edP1wJ2zOBpfCyhIS7g==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@vitest/pretty-format": "3.2.7",
+        "magic-string": "^0.30.17",
+        "pathe": "^2.0.3"
+      },
+      "funding": {
+        "url": "https://opencollective.com/vitest"
+      }
+    },
+    "node_modules/@vitest/spy": {
+      "version": "3.2.7",
+      "resolved": "https://registry.npmjs.org/@vitest/spy/-/spy-3.2.7.tgz",
+      "integrity": "sha512-Q2eQGI6d2L/hBtZ0qNuKcAGid68XK6cv1xsoaIma6PaJhHPoqcEJhYpXZ/5myCMqkNgtP6UKuBhbc0nHKnrkuQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "tinyspy": "^4.0.3"
+      },
+      "funding": {
+        "url": "https://opencollective.com/vitest"
+      }
+    },
+    "node_modules/@vitest/utils": {
+      "version": "3.2.7",
+      "resolved": "https://registry.npmjs.org/@vitest/utils/-/utils-3.2.7.tgz",
+      "integrity": "sha512-x6BDOd7dyo3PFLY3I9/HJ25X/6OurhGXk2/B9gOZNPF7XDVjeBK4k01lQE5uvDpbuheErh91qYuE1E2OEjK3Rw==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@vitest/pretty-format": "3.2.7",
+        "loupe": "^3.1.4",
+        "tinyrainbow": "^2.0.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/vitest"
+      }
+    },
     "node_modules/acorn": {
       "version": "8.16.0",
       "resolved": "https://registry.npmjs.org/acorn/-/acorn-8.16.0.tgz",
@@ -2621,6 +3499,16 @@
         "acorn": "^6.0.0 || ^7.0.0 || ^8.0.0"
       }
     },
+    "node_modules/agent-base": {
+      "version": "7.1.4",
+      "resolved": "https://registry.npmjs.org/agent-base/-/agent-base-7.1.4.tgz",
+      "integrity": "sha512-MnA+YT8fwfJPgBx3m60MNqakm30XOkyIoH1y6huTQvC0PwZG7ki8NacLBcrPbNoo8vEZy7Jpuk7+jMO+CUovTQ==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">= 14"
+      }
+    },
     "node_modules/ajv": {
       "version": "6.15.0",
       "resolved": "https://registry.npmjs.org/ajv/-/ajv-6.15.0.tgz",
@@ -2638,6 +3526,17 @@
         "url": "https://github.com/sponsors/epoberezkin"
       }
     },
+    "node_modules/ansi-regex": {
+      "version": "5.0.1",
+      "resolved": "https://registry.npmjs.org/ansi-regex/-/ansi-regex-5.0.1.tgz",
+      "integrity": "sha512-quJQXlTSUGL2LH9SUXo8VwsY4soanhgo6LNSm84E1LBcE8s3O0wpdiRzyR9z/ZZJMlMWv37qOOb9pdJlMUEKFQ==",
+      "dev": true,
+      "license": "MIT",
+      "peer": true,
+      "engines": {
+        "node": ">=8"
+      }
+    },
     "node_modules/ansi-styles": {
       "version": "4.3.0",
       "resolved": "https://registry.npmjs.org/ansi-styles/-/ansi-styles-4.3.0.tgz",
@@ -2825,10 +3724,20 @@
         "is-array-buffer": "^3.0.4"
       },
       "engines": {
-        "node": ">= 0.4"
-      },
-      "funding": {
-        "url": "https://github.com/sponsors/ljharb"
+        "node": ">= 0.4"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/ljharb"
+      }
+    },
+    "node_modules/assertion-error": {
+      "version": "2.0.1",
+      "resolved": "https://registry.npmjs.org/assertion-error/-/assertion-error-2.0.1.tgz",
+      "integrity": "sha512-Izi8RQcffqCeNVgFigKli1ssklIbpHnCYc6AknXGYoB6grJqyeby7jv12JUQgmTAnIDnbck1uxksT4dzN3PWBA==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=12"
       }
     },
     "node_modules/ast-types-flow": {
@@ -2903,6 +3812,16 @@
         "node": ">=6.0.0"
       }
     },
+    "node_modules/bidi-js": {
+      "version": "1.0.3",
+      "resolved": "https://registry.npmjs.org/bidi-js/-/bidi-js-1.0.3.tgz",
+      "integrity": "sha512-RKshQI1R3YQ+n9YJz2QQ147P66ELpa1FQEg20Dk8oW9t2KgLbpDLLp9aGZ7y8WHSshDknG0bknqGw5/tyCs5tw==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "require-from-string": "^2.0.2"
+      }
+    },
     "node_modules/brace-expansion": {
       "version": "1.1.14",
       "resolved": "https://registry.npmjs.org/brace-expansion/-/brace-expansion-1.1.14.tgz",
@@ -2961,6 +3880,16 @@
         "node": "^6 || ^7 || ^8 || ^9 || ^10 || ^11 || ^12 || >=13.7"
       }
     },
+    "node_modules/cac": {
+      "version": "6.7.14",
+      "resolved": "https://registry.npmjs.org/cac/-/cac-6.7.14.tgz",
+      "integrity": "sha512-b6Ilus+c3RrdDk+JhLKUAQfzzgLEPy6wcXqS7f/xe1EETvsDP6GORG7SFuOs6cID5YkqchW/LXZbX5bc8j7ZcQ==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=8"
+      }
+    },
     "node_modules/call-bind": {
       "version": "1.0.9",
       "resolved": "https://registry.npmjs.org/call-bind/-/call-bind-1.0.9.tgz",
@@ -3041,6 +3970,23 @@
       ],
       "license": "CC-BY-4.0"
     },
+    "node_modules/chai": {
+      "version": "5.3.3",
+      "resolved": "https://registry.npmjs.org/chai/-/chai-5.3.3.tgz",
+      "integrity": "sha512-4zNhdJD/iOjSH0A05ea+Ke6MU5mmpQcbQsSOkgdaUMJ9zTlDTD/GYlwohmIE2u0gaxHYiVHEn1Fw9mZ/ktJWgw==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "assertion-error": "^2.0.1",
+        "check-error": "^2.1.1",
+        "deep-eql": "^5.0.1",
+        "loupe": "^3.1.0",
+        "pathval": "^2.0.0"
+      },
+      "engines": {
+        "node": ">=18"
+      }
+    },
     "node_modules/chalk": {
       "version": "4.1.2",
       "resolved": "https://registry.npmjs.org/chalk/-/chalk-4.1.2.tgz",
@@ -3058,6 +4004,16 @@
         "url": "https://github.com/chalk/chalk?sponsor=1"
       }
     },
+    "node_modules/check-error": {
+      "version": "2.1.3",
+      "resolved": "https://registry.npmjs.org/check-error/-/check-error-2.1.3.tgz",
+      "integrity": "sha512-PAJdDJusoxnwm1VwW07VWwUN1sl7smmC3OKggvndJFadxxDRyFJBX/ggnu/KE4kQAB7a3Dp8f/YXC1FlUprWmA==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">= 16"
+      }
+    },
     "node_modules/client-only": {
       "version": "0.0.1",
       "resolved": "https://registry.npmjs.org/client-only/-/client-only-0.0.1.tgz",
@@ -3113,6 +4069,53 @@
         "node": ">= 8"
       }
     },
+    "node_modules/css-tree": {
+      "version": "3.2.1",
+      "resolved": "https://registry.npmjs.org/css-tree/-/css-tree-3.2.1.tgz",
+      "integrity": "sha512-X7sjQzceUhu1u7Y/ylrRZFU2FS6LRiFVp6rKLPg23y3x3c3DOKAwuXGDp+PAGjh6CSnCjYeAul8pcT8bAl+lSA==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "mdn-data": "2.27.1",
+        "source-map-js": "^1.2.1"
+      },
+      "engines": {
+        "node": "^10 || ^12.20.0 || ^14.13.0 || >=15.0.0"
+      }
+    },
+    "node_modules/css.escape": {
+      "version": "1.5.1",
+      "resolved": "https://registry.npmjs.org/css.escape/-/css.escape-1.5.1.tgz",
+      "integrity": "sha512-YUifsXXuknHlUsmlgyY0PKzgPOr7/FjCePfHNt0jxm83wHZi44VDMQ7/fGNkjY3/jV1MC+1CmZbaHzugyeRtpg==",
+      "dev": true,
+      "license": "MIT"
+    },
+    "node_modules/cssstyle": {
+      "version": "5.3.7",
+      "resolved": "https://registry.npmjs.org/cssstyle/-/cssstyle-5.3.7.tgz",
+      "integrity": "sha512-7D2EPVltRrsTkhpQmksIu+LxeWAIEk6wRDMJ1qljlv+CKHJM+cJLlfhWIzNA44eAsHXSNe3+vO6DW1yCYx8SuQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@asamuzakjp/css-color": "^4.1.1",
+        "@csstools/css-syntax-patches-for-csstree": "^1.0.21",
+        "css-tree": "^3.1.0",
+        "lru-cache": "^11.2.4"
+      },
+      "engines": {
+        "node": ">=20"
+      }
+    },
+    "node_modules/cssstyle/node_modules/lru-cache": {
+      "version": "11.5.2",
+      "resolved": "https://registry.npmjs.org/lru-cache/-/lru-cache-11.5.2.tgz",
+      "integrity": "sha512-4pfM1Ff0x50o0tQwb5ucw/RzNyD0/YJME6IVcStalZuMWxdt3sR3huStTtxz4PUmvZfRguvDejasvQ2kifR11g==",
+      "dev": true,
+      "license": "BlueOak-1.0.0",
+      "engines": {
+        "node": "20 || >=22"
+      }
+    },
     "node_modules/csstype": {
       "version": "3.2.3",
       "resolved": "https://registry.npmjs.org/csstype/-/csstype-3.2.3.tgz",
@@ -3127,6 +4130,30 @@
       "dev": true,
       "license": "BSD-2-Clause"
     },
+    "node_modules/data-urls": {
+      "version": "6.0.1",
+      "resolved": "https://registry.npmjs.org/data-urls/-/data-urls-6.0.1.tgz",
+      "integrity": "sha512-euIQENZg6x8mj3fO6o9+fOW8MimUI4PpD/fZBhJfeioZVy9TUpM4UY7KjQNVZFlqwJ0UdzRDzkycB997HEq1BQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "whatwg-mimetype": "^5.0.0",
+        "whatwg-url": "^15.1.0"
+      },
+      "engines": {
+        "node": ">=20"
+      }
+    },
+    "node_modules/data-urls/node_modules/whatwg-mimetype": {
+      "version": "5.0.0",
+      "resolved": "https://registry.npmjs.org/whatwg-mimetype/-/whatwg-mimetype-5.0.0.tgz",
+      "integrity": "sha512-sXcNcHOC51uPGF0P/D4NVtrkjSU2fNsm9iog4ZvZJsL3rjoDAzXZhkm2MWt1y+PUdggKAYVoMAIYcs78wJ51Cw==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=20"
+      }
+    },
     "node_modules/data-view-buffer": {
       "version": "1.0.2",
       "resolved": "https://registry.npmjs.org/data-view-buffer/-/data-view-buffer-1.0.2.tgz",
@@ -3199,6 +4226,23 @@
         }
       }
     },
+    "node_modules/decimal.js": {
+      "version": "10.6.0",
+      "resolved": "https://registry.npmjs.org/decimal.js/-/decimal.js-10.6.0.tgz",
+      "integrity": "sha512-YpgQiITW3JXGntzdUmyUR1V812Hn8T1YVXhCu+wO3OpS4eU9l4YdD3qjyiKdV6mvV29zapkMeD390UVEf2lkUg==",
+      "dev": true,
+      "license": "MIT"
+    },
+    "node_modules/deep-eql": {
+      "version": "5.0.2",
+      "resolved": "https://registry.npmjs.org/deep-eql/-/deep-eql-5.0.2.tgz",
+      "integrity": "sha512-h5k/5U50IJJFpzfL6nO9jaaumfjO/f2NjK/oYB2Djzm4p9L+3T9qWpZqZ2hAbLPuuYq9wrU08WQyBTL5GbPk5Q==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=6"
+      }
+    },
     "node_modules/deep-is": {
       "version": "0.1.4",
       "resolved": "https://registry.npmjs.org/deep-is/-/deep-is-0.1.4.tgz",
@@ -3242,6 +4286,17 @@
         "url": "https://github.com/sponsors/ljharb"
       }
     },
+    "node_modules/dequal": {
+      "version": "2.0.3",
+      "resolved": "https://registry.npmjs.org/dequal/-/dequal-2.0.3.tgz",
+      "integrity": "sha512-0je+qPKHEMohvfRTCEo3CrPG6cAzAYgmzKyxRiYSSDkS6eGJdyVJm7WaYA5ECaAD9wLB2T4EEeymA5aFVcYXCA==",
+      "dev": true,
+      "license": "MIT",
+      "peer": true,
+      "engines": {
+        "node": ">=6"
+      }
+    },
     "node_modules/detect-libc": {
       "version": "2.1.2",
       "resolved": "https://registry.npmjs.org/detect-libc/-/detect-libc-2.1.2.tgz",
@@ -3265,6 +4320,14 @@
         "node": ">=0.10.0"
       }
     },
+    "node_modules/dom-accessibility-api": {
+      "version": "0.5.16",
+      "resolved": "https://registry.npmjs.org/dom-accessibility-api/-/dom-accessibility-api-0.5.16.tgz",
+      "integrity": "sha512-X7BJ2yElsnOJ30pZF4uIIDfBEVgF4XEBxL9Bxhy6dnrm5hkzqmsWHGTiHqRiITNhMyFLyAiWndIJP7Z1NTteDg==",
+      "dev": true,
+      "license": "MIT",
+      "peer": true
+    },
     "node_modules/dunder-proto": {
       "version": "1.0.1",
       "resolved": "https://registry.npmjs.org/dunder-proto/-/dunder-proto-1.0.1.tgz",
@@ -3308,6 +4371,19 @@
         "node": ">=10.13.0"
       }
     },
+    "node_modules/entities": {
+      "version": "8.0.0",
+      "resolved": "https://registry.npmjs.org/entities/-/entities-8.0.0.tgz",
+      "integrity": "sha512-zwfzJecQ/Uej6tusMqwAqU/6KL2XaB2VZ2Jg54Je6ahNBGNH6Ek6g3jjNCF0fG9EWQKGZNddNjU5F1ZQn/sBnA==",
+      "dev": true,
+      "license": "BSD-2-Clause",
+      "engines": {
+        "node": ">=20.19.0"
+      },
+      "funding": {
+        "url": "https://github.com/fb55/entities?sponsor=1"
+      }
+    },
     "node_modules/es-abstract": {
       "version": "1.24.2",
       "resolved": "https://registry.npmjs.org/es-abstract/-/es-abstract-1.24.2.tgz",
@@ -3425,6 +4501,13 @@
         "node": ">= 0.4"
       }
     },
+    "node_modules/es-module-lexer": {
+      "version": "1.7.0",
+      "resolved": "https://registry.npmjs.org/es-module-lexer/-/es-module-lexer-1.7.0.tgz",
+      "integrity": "sha512-jEQoCwk8hyb2AZziIOLhDqpm5+2ww5uIE6lkO/6jcOCusfk6LhMHpXXfBLXTZ7Ydyt0j4VoUQv6uGNYbdW+kBA==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/es-object-atoms": {
       "version": "1.1.1",
       "resolved": "https://registry.npmjs.org/es-object-atoms/-/es-object-atoms-1.1.1.tgz",
@@ -3946,6 +5029,16 @@
         "node": ">=4.0"
       }
     },
+    "node_modules/estree-walker": {
+      "version": "3.0.3",
+      "resolved": "https://registry.npmjs.org/estree-walker/-/estree-walker-3.0.3.tgz",
+      "integrity": "sha512-7RUKfXgSMMkzt6ZuXmqapOurLGPPfgj6l9uRZ7lRGolvk0y2yocc35LdcxKC5PQZdn2DMqioAQ2NoWcrTKmm6g==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@types/estree": "^1.0.0"
+      }
+    },
     "node_modules/esutils": {
       "version": "2.0.3",
       "resolved": "https://registry.npmjs.org/esutils/-/esutils-2.0.3.tgz",
@@ -3956,6 +5049,16 @@
         "node": ">=0.10.0"
       }
     },
+    "node_modules/expect-type": {
+      "version": "1.4.0",
+      "resolved": "https://registry.npmjs.org/expect-type/-/expect-type-1.4.0.tgz",
+      "integrity": "sha512-KfYbmpRm0VbLjEvVa9yGwCi9GI34xvi7A/HXYWQO65CSD2u3MczUJSuwXKFIxlGsgBQizV9q5J9NHj4VG0n+pA==",
+      "dev": true,
+      "license": "Apache-2.0",
+      "engines": {
+        "node": ">=12.0.0"
+      }
+    },
     "node_modules/fast-deep-equal": {
       "version": "3.1.3",
       "resolved": "https://registry.npmjs.org/fast-deep-equal/-/fast-deep-equal-3.1.3.tgz",
@@ -4417,6 +5520,47 @@
         "hermes-estree": "0.25.1"
       }
     },
+    "node_modules/html-encoding-sniffer": {
+      "version": "6.0.0",
+      "resolved": "https://registry.npmjs.org/html-encoding-sniffer/-/html-encoding-sniffer-6.0.0.tgz",
+      "integrity": "sha512-CV9TW3Y3f8/wT0BRFc1/KAVQ3TUHiXmaAb6VW9vtiMFf7SLoMd1PdAc4W3KFOFETBJUb90KatHqlsZMWV+R9Gg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@exodus/bytes": "^1.6.0"
+      },
+      "engines": {
+        "node": "^20.19.0 || ^22.12.0 || >=24.0.0"
+      }
+    },
+    "node_modules/http-proxy-agent": {
+      "version": "7.0.2",
+      "resolved": "https://registry.npmjs.org/http-proxy-agent/-/http-proxy-agent-7.0.2.tgz",
+      "integrity": "sha512-T1gkAiYYDWYx3V5Bmyu7HcfcvL7mUrTWiM6yOfa3PIphViJ/gFPbvidQ+veqSOHci/PxBcDabeUNCzpOODJZig==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "agent-base": "^7.1.0",
+        "debug": "^4.3.4"
+      },
+      "engines": {
+        "node": ">= 14"
+      }
+    },
+    "node_modules/https-proxy-agent": {
+      "version": "7.0.6",
+      "resolved": "https://registry.npmjs.org/https-proxy-agent/-/https-proxy-agent-7.0.6.tgz",
+      "integrity": "sha512-vK9P5/iUfdl95AI+JVyUuIcVtd4ofvtrOr3HNtM2yxC9bnMbEdp3x01OhQNnjb8IJYi38VlTE3mBXwcfvywuSw==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "agent-base": "^7.1.2",
+        "debug": "4"
+      },
+      "engines": {
+        "node": ">= 14"
+      }
+    },
     "node_modules/ignore": {
       "version": "5.3.2",
       "resolved": "https://registry.npmjs.org/ignore/-/ignore-5.3.2.tgz",
@@ -4454,6 +5598,16 @@
         "node": ">=0.8.19"
       }
     },
+    "node_modules/indent-string": {
+      "version": "4.0.0",
+      "resolved": "https://registry.npmjs.org/indent-string/-/indent-string-4.0.0.tgz",
+      "integrity": "sha512-EdDDZu4A2OyIK7Lr/2zG+w5jmbuk1DVBnEwREQvBzspBJkCEbRa8GxU1lghYcaGJCnRWibjDXlq779X1/y5xwg==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=8"
+      }
+    },
     "node_modules/internal-slot": {
       "version": "1.1.0",
       "resolved": "https://registry.npmjs.org/internal-slot/-/internal-slot-1.1.0.tgz",
@@ -4739,6 +5893,13 @@
         "url": "https://github.com/sponsors/ljharb"
       }
     },
+    "node_modules/is-potential-custom-element-name": {
+      "version": "1.0.1",
+      "resolved": "https://registry.npmjs.org/is-potential-custom-element-name/-/is-potential-custom-element-name-1.0.1.tgz",
+      "integrity": "sha512-bCYeRA2rVibKZd+s2625gGnGF/t7DSqDs4dP7CrLA1m7jKWz6pps0LpYLJN8Q64HtmPKJ1hrN3nzPNKFEKOUiQ==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/is-regex": {
       "version": "1.2.1",
       "resolved": "https://registry.npmjs.org/is-regex/-/is-regex-1.2.1.tgz",
@@ -4946,6 +6107,46 @@
         "js-yaml": "bin/js-yaml.js"
       }
     },
+    "node_modules/jsdom": {
+      "version": "27.4.0",
+      "resolved": "https://registry.npmjs.org/jsdom/-/jsdom-27.4.0.tgz",
+      "integrity": "sha512-mjzqwWRD9Y1J1KUi7W97Gja1bwOOM5Ug0EZ6UDK3xS7j7mndrkwozHtSblfomlzyB4NepioNt+B2sOSzczVgtQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@acemir/cssom": "^0.9.28",
+        "@asamuzakjp/dom-selector": "^6.7.6",
+        "@exodus/bytes": "^1.6.0",
+        "cssstyle": "^5.3.4",
+        "data-urls": "^6.0.0",
+        "decimal.js": "^10.6.0",
+        "html-encoding-sniffer": "^6.0.0",
+        "http-proxy-agent": "^7.0.2",
+        "https-proxy-agent": "^7.0.6",
+        "is-potential-custom-element-name": "^1.0.1",
+        "parse5": "^8.0.0",
+        "saxes": "^6.0.0",
+        "symbol-tree": "^3.2.4",
+        "tough-cookie": "^6.0.0",
+        "w3c-xmlserializer": "^5.0.0",
+        "webidl-conversions": "^8.0.0",
+        "whatwg-mimetype": "^4.0.0",
+        "whatwg-url": "^15.1.0",
+        "ws": "^8.18.3",
+        "xml-name-validator": "^5.0.0"
+      },
+      "engines": {
+        "node": "^20.19.0 || ^22.12.0 || >=24.0.0"
+      },
+      "peerDependencies": {
+        "canvas": "^3.0.0"
+      },
+      "peerDependenciesMeta": {
+        "canvas": {
+          "optional": true
+        }
+      }
+    },
     "node_modules/jsesc": {
       "version": "3.1.0",
       "resolved": "https://registry.npmjs.org/jsesc/-/jsesc-3.1.0.tgz",
@@ -5350,6 +6551,13 @@
         "loose-envify": "cli.js"
       }
     },
+    "node_modules/loupe": {
+      "version": "3.2.1",
+      "resolved": "https://registry.npmjs.org/loupe/-/loupe-3.2.1.tgz",
+      "integrity": "sha512-CdzqowRJCeLU72bHvWqwRBBlLcMEtIvGrlvef74kMnV2AolS9Y8xUv1I0U/MNAWMhBlKIoyuEgoJ0t/bbwHbLQ==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/lru-cache": {
       "version": "5.1.1",
       "resolved": "https://registry.npmjs.org/lru-cache/-/lru-cache-5.1.1.tgz",
@@ -5360,6 +6568,17 @@
         "yallist": "^3.0.2"
       }
     },
+    "node_modules/lz-string": {
+      "version": "1.5.0",
+      "resolved": "https://registry.npmjs.org/lz-string/-/lz-string-1.5.0.tgz",
+      "integrity": "sha512-h5bgJWpxJNswbU7qCrV0tIKQCaS3blPDrqKWx+QxzuzL1zGUzij9XCWLrSLsJPu5t+eWA/ycetzYAO5IOMcWAQ==",
+      "dev": true,
+      "license": "MIT",
+      "peer": true,
+      "bin": {
+        "lz-string": "bin/bin.js"
+      }
+    },
     "node_modules/magic-string": {
       "version": "0.30.21",
       "resolved": "https://registry.npmjs.org/magic-string/-/magic-string-0.30.21.tgz",
@@ -5380,6 +6599,13 @@
         "node": ">= 0.4"
       }
     },
+    "node_modules/mdn-data": {
+      "version": "2.27.1",
+      "resolved": "https://registry.npmjs.org/mdn-data/-/mdn-data-2.27.1.tgz",
+      "integrity": "sha512-9Yubnt3e8A0OKwxYSXyhLymGW4sCufcLG6VdiDdUGVkPhpqLxlvP5vl1983gQjJl3tqbrM731mjaZaP68AgosQ==",
+      "dev": true,
+      "license": "CC0-1.0"
+    },
     "node_modules/merge2": {
       "version": "1.4.1",
       "resolved": "https://registry.npmjs.org/merge2/-/merge2-1.4.1.tgz",
@@ -5404,6 +6630,16 @@
         "node": ">=8.6"
       }
     },
+    "node_modules/min-indent": {
+      "version": "1.0.1",
+      "resolved": "https://registry.npmjs.org/min-indent/-/min-indent-1.0.1.tgz",
+      "integrity": "sha512-I9jwMn07Sy/IwOj3zVkVik2JTvgpaykDZEigL6Rx6N9LbMywwUSMtxET+7lVoDLLd3O3IXwJwvuuns8UB/HeAg==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=4"
+      }
+    },
     "node_modules/minimatch": {
       "version": "3.1.5",
       "resolved": "https://registry.npmjs.org/minimatch/-/minimatch-3.1.5.tgz",
@@ -5786,6 +7022,19 @@
         "node": ">=6"
       }
     },
+    "node_modules/parse5": {
+      "version": "8.0.1",
+      "resolved": "https://registry.npmjs.org/parse5/-/parse5-8.0.1.tgz",
+      "integrity": "sha512-z1e/HMG90obSGeidlli3hj7cbocou0/wa5HacvI3ASx34PecNjNQeaHNo5WIZpWofN9kgkqV1q5YvXe3F0FoPw==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "entities": "^8.0.0"
+      },
+      "funding": {
+        "url": "https://github.com/inikulin/parse5?sponsor=1"
+      }
+    },
     "node_modules/path-exists": {
       "version": "4.0.0",
       "resolved": "https://registry.npmjs.org/path-exists/-/path-exists-4.0.0.tgz",
@@ -5813,6 +7062,23 @@
       "dev": true,
       "license": "MIT"
     },
+    "node_modules/pathe": {
+      "version": "2.0.3",
+      "resolved": "https://registry.npmjs.org/pathe/-/pathe-2.0.3.tgz",
+      "integrity": "sha512-WUjGcAqP1gQacoQe+OBJsFA7Ld4DyXuUIjZ5cc75cLHvJ7dtNsTugphxIADwspS+AraAUePCKrSVtPLFj/F88w==",
+      "dev": true,
+      "license": "MIT"
+    },
+    "node_modules/pathval": {
+      "version": "2.0.1",
+      "resolved": "https://registry.npmjs.org/pathval/-/pathval-2.0.1.tgz",
+      "integrity": "sha512-//nshmD55c46FuFw26xV/xFAaB5HF9Xdap7HJBBnrKdAd6/GxDBaNA1870O79+9ueg61cZLSVc+OaFlfmObYVQ==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">= 14.16"
+      }
+    },
     "node_modules/picocolors": {
       "version": "1.1.1",
       "resolved": "https://registry.npmjs.org/picocolors/-/picocolors-1.1.1.tgz",
@@ -5881,6 +7147,44 @@
         "node": ">= 0.8.0"
       }
     },
+    "node_modules/pretty-format": {
+      "version": "27.5.1",
+      "resolved": "https://registry.npmjs.org/pretty-format/-/pretty-format-27.5.1.tgz",
+      "integrity": "sha512-Qb1gy5OrP5+zDf2Bvnzdl3jsTf1qXVMazbvCoKhtKqVs4/YK4ozX4gKQJJVyNe+cajNPn0KoC0MC3FUmaHWEmQ==",
+      "dev": true,
+      "license": "MIT",
+      "peer": true,
+      "dependencies": {
+        "ansi-regex": "^5.0.1",
+        "ansi-styles": "^5.0.0",
+        "react-is": "^17.0.1"
+      },
+      "engines": {
+        "node": "^10.13.0 || ^12.13.0 || ^14.15.0 || >=15.0.0"
+      }
+    },
+    "node_modules/pretty-format/node_modules/ansi-styles": {
+      "version": "5.2.0",
+      "resolved": "https://registry.npmjs.org/ansi-styles/-/ansi-styles-5.2.0.tgz",
+      "integrity": "sha512-Cxwpt2SfTzTtXcfOlzGEee8O+c+MmUgGrNiBcXnuWxuFJHe6a5Hz7qwhwe5OgaSYI0IJvkLqWX1ASG+cJOkEiA==",
+      "dev": true,
+      "license": "MIT",
+      "peer": true,
+      "engines": {
+        "node": ">=10"
+      },
+      "funding": {
+        "url": "https://github.com/chalk/ansi-styles?sponsor=1"
+      }
+    },
+    "node_modules/pretty-format/node_modules/react-is": {
+      "version": "17.0.2",
+      "resolved": "https://registry.npmjs.org/react-is/-/react-is-17.0.2.tgz",
+      "integrity": "sha512-w2GsyukL62IJnlaff/nRegPQR94C/XXamvMWmSHRJ4y7Ts/4ocGRmTHvOs8PSE6pB3dWOrD/nueuU5sduBsQ4w==",
+      "dev": true,
+      "license": "MIT",
+      "peer": true
+    },
     "node_modules/prop-types": {
       "version": "15.8.1",
       "resolved": "https://registry.npmjs.org/prop-types/-/prop-types-15.8.1.tgz",
@@ -5952,6 +7256,20 @@
       "dev": true,
       "license": "MIT"
     },
+    "node_modules/redent": {
+      "version": "3.0.0",
+      "resolved": "https://registry.npmjs.org/redent/-/redent-3.0.0.tgz",
+      "integrity": "sha512-6tDA8g98We0zd0GvVeMT9arEOnTw9qM03L9cJXaCjrip1OO764RDBLBfrB4cwzNGDj5OA5ioymC9GkizgWJDUg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "indent-string": "^4.0.0",
+        "strip-indent": "^3.0.0"
+      },
+      "engines": {
+        "node": ">=8"
+      }
+    },
     "node_modules/reflect.getprototypeof": {
       "version": "1.0.10",
       "resolved": "https://registry.npmjs.org/reflect.getprototypeof/-/reflect.getprototypeof-1.0.10.tgz",
@@ -5996,6 +7314,16 @@
         "url": "https://github.com/sponsors/ljharb"
       }
     },
+    "node_modules/require-from-string": {
+      "version": "2.0.2",
+      "resolved": "https://registry.npmjs.org/require-from-string/-/require-from-string-2.0.2.tgz",
+      "integrity": "sha512-Xf0nWe6RseziFMu+Ap9biiUbmplq6S9/p+7w7YXP/JBHhrUDDUhwa+vANyubuqfZWTveU//DYVGsDG7RKL/vEw==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=0.10.0"
+      }
+    },
     "node_modules/resolve": {
       "version": "2.0.0-next.6",
       "resolved": "https://registry.npmjs.org/resolve/-/resolve-2.0.0-next.6.tgz",
@@ -6051,6 +7379,52 @@
         "node": ">=0.10.0"
       }
     },
+    "node_modules/rollup": {
+      "version": "4.62.4",
+      "resolved": "https://registry.npmjs.org/rollup/-/rollup-4.62.4.tgz",
+      "integrity": "sha512-RXOqwaPsBGjMNMa4sQjDjHieHEZDFoj/Rdr46l2MU5DfEs16wHJPC2RPTPHWhNl+M3aI472LLqFkFKut4SblOg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@types/estree": "1.0.9"
+      },
+      "bin": {
+        "rollup": "dist/bin/rollup"
+      },
+      "engines": {
+        "node": ">=18.0.0",
+        "npm": ">=8.0.0"
+      },
+      "optionalDependencies": {
+        "@napi-rs/lzma-linux-x64-gnu": "1.5.1",
+        "@rollup/rollup-android-arm-eabi": "4.62.4",
+        "@rollup/rollup-android-arm64": "4.62.4",
+        "@rollup/rollup-darwin-arm64": "4.62.4",
+        "@rollup/rollup-darwin-x64": "4.62.4",
+        "@rollup/rollup-freebsd-arm64": "4.62.4",
+        "@rollup/rollup-freebsd-x64": "4.62.4",
+        "@rollup/rollup-linux-arm-gnueabihf": "4.62.4",
+        "@rollup/rollup-linux-arm-musleabihf": "4.62.4",
+        "@rollup/rollup-linux-arm64-gnu": "4.62.4",
+        "@rollup/rollup-linux-arm64-musl": "4.62.4",
+        "@rollup/rollup-linux-loong64-gnu": "4.62.4",
+        "@rollup/rollup-linux-loong64-musl": "4.62.4",
+        "@rollup/rollup-linux-ppc64-gnu": "4.62.4",
+        "@rollup/rollup-linux-ppc64-musl": "4.62.4",
+        "@rollup/rollup-linux-riscv64-gnu": "4.62.4",
+        "@rollup/rollup-linux-riscv64-musl": "4.62.4",
+        "@rollup/rollup-linux-s390x-gnu": "4.62.4",
+        "@rollup/rollup-linux-x64-gnu": "4.62.4",
+        "@rollup/rollup-linux-x64-musl": "4.62.4",
+        "@rollup/rollup-openbsd-x64": "4.62.4",
+        "@rollup/rollup-openharmony-arm64": "4.62.4",
+        "@rollup/rollup-win32-arm64-msvc": "4.62.4",
+        "@rollup/rollup-win32-ia32-msvc": "4.62.4",
+        "@rollup/rollup-win32-x64-gnu": "4.62.4",
+        "@rollup/rollup-win32-x64-msvc": "4.62.4",
+        "fsevents": "~2.3.2"
+      }
+    },
     "node_modules/run-parallel": {
       "version": "1.2.0",
       "resolved": "https://registry.npmjs.org/run-parallel/-/run-parallel-1.2.0.tgz",
@@ -6130,6 +7504,19 @@
         "url": "https://github.com/sponsors/ljharb"
       }
     },
+    "node_modules/saxes": {
+      "version": "6.0.0",
+      "resolved": "https://registry.npmjs.org/saxes/-/saxes-6.0.0.tgz",
+      "integrity": "sha512-xAg7SOnEhrm5zI3puOOKyy1OMcMlIJZYNJY7xLBwSze0UjhPLnWfj2GF2EpT0jmzaJKIWKHLsaSSajf35bcYnA==",
+      "dev": true,
+      "license": "ISC",
+      "dependencies": {
+        "xmlchars": "^2.2.0"
+      },
+      "engines": {
+        "node": ">=v12.22.7"
+      }
+    },
     "node_modules/scheduler": {
       "version": "0.27.0",
       "resolved": "https://registry.npmjs.org/scheduler/-/scheduler-0.27.0.tgz",
@@ -6352,6 +7739,13 @@
         "url": "https://github.com/sponsors/ljharb"
       }
     },
+    "node_modules/siginfo": {
+      "version": "2.0.0",
+      "resolved": "https://registry.npmjs.org/siginfo/-/siginfo-2.0.0.tgz",
+      "integrity": "sha512-ybx0WO1/8bSBLEWXZvEd7gMW3Sn3JFlW3TvX1nREbDLRNQNaeNN8WK0meBwPdAaOI7TtRRRJn/Es1zhrrCHu7g==",
+      "dev": true,
+      "license": "ISC"
+    },
     "node_modules/simple-icons": {
       "version": "16.18.1",
       "resolved": "https://registry.npmjs.org/simple-icons/-/simple-icons-16.18.1.tgz",
@@ -6387,6 +7781,20 @@
       "dev": true,
       "license": "MIT"
     },
+    "node_modules/stackback": {
+      "version": "0.0.2",
+      "resolved": "https://registry.npmjs.org/stackback/-/stackback-0.0.2.tgz",
+      "integrity": "sha512-1XMJE5fQo1jGH6Y/7ebnwPOBEkIEnT4QF32d5R1+VXdXveM0IBMJt8zfaxX1P3QhVwrYe+576+jkANtSS2mBbw==",
+      "dev": true,
+      "license": "MIT"
+    },
+    "node_modules/std-env": {
+      "version": "3.10.0",
+      "resolved": "https://registry.npmjs.org/std-env/-/std-env-3.10.0.tgz",
+      "integrity": "sha512-5GS12FdOZNliM5mAOxFRg7Ir0pWz8MdpYm6AY6VPkGpbA7ZzmbzNcBJQ0GPvvyWgcY7QAhCgf9Uy89I03faLkg==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/stop-iteration-iterator": {
       "version": "1.1.0",
       "resolved": "https://registry.npmjs.org/stop-iteration-iterator/-/stop-iteration-iterator-1.1.0.tgz",
@@ -6524,6 +7932,19 @@
         "node": ">=4"
       }
     },
+    "node_modules/strip-indent": {
+      "version": "3.0.0",
+      "resolved": "https://registry.npmjs.org/strip-indent/-/strip-indent-3.0.0.tgz",
+      "integrity": "sha512-laJTa3Jb+VQpaC6DseHhF7dXVqHTfJPCRDaEbid/drOhgitgYku/letMUqOXFoWV0zIIUbjpdH2t+tYj4bQMRQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "min-indent": "^1.0.0"
+      },
+      "engines": {
+        "node": ">=8"
+      }
+    },
     "node_modules/strip-json-comments": {
       "version": "3.1.1",
       "resolved": "https://registry.npmjs.org/strip-json-comments/-/strip-json-comments-3.1.1.tgz",
@@ -6537,6 +7958,26 @@
         "url": "https://github.com/sponsors/sindresorhus"
       }
     },
+    "node_modules/strip-literal": {
+      "version": "3.1.0",
+      "resolved": "https://registry.npmjs.org/strip-literal/-/strip-literal-3.1.0.tgz",
+      "integrity": "sha512-8r3mkIM/2+PpjHoOtiAW8Rg3jJLHaV7xPwG+YRGrv6FP0wwk/toTpATxWYOW0BKdWwl82VT2tFYi5DlROa0Mxg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "js-tokens": "^9.0.1"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/antfu"
+      }
+    },
+    "node_modules/strip-literal/node_modules/js-tokens": {
+      "version": "9.0.1",
+      "resolved": "https://registry.npmjs.org/js-tokens/-/js-tokens-9.0.1.tgz",
+      "integrity": "sha512-mxa9E9ITFOt0ban3j6L5MpjwegGz6lBQmM1IJkWeBZGcMxto50+eWdjC/52xDbS2vy0k7vIMK0Fe2wfL9OQSpQ==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/styled-jsx": {
       "version": "5.1.6",
       "resolved": "https://registry.npmjs.org/styled-jsx/-/styled-jsx-5.1.6.tgz",
@@ -6586,6 +8027,13 @@
         "url": "https://github.com/sponsors/ljharb"
       }
     },
+    "node_modules/symbol-tree": {
+      "version": "3.2.4",
+      "resolved": "https://registry.npmjs.org/symbol-tree/-/symbol-tree-3.2.4.tgz",
+      "integrity": "sha512-9QNk5KwDF+Bvz+PyObkmSYjI5ksVUYtjW7AU22r2NKcfLJcXp96hkDWU3+XndOsUb+AQ9QhfzfCT2O+CNWT5Tw==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/tailwindcss": {
       "version": "4.2.4",
       "resolved": "https://registry.npmjs.org/tailwindcss/-/tailwindcss-4.2.4.tgz",
@@ -6607,6 +8055,20 @@
         "url": "https://opencollective.com/webpack"
       }
     },
+    "node_modules/tinybench": {
+      "version": "2.9.0",
+      "resolved": "https://registry.npmjs.org/tinybench/-/tinybench-2.9.0.tgz",
+      "integrity": "sha512-0+DUvqWMValLmha6lr4kD8iAMK1HzV0/aKnCtWb9v9641TnP/MFb7Pc2bxoxQjTXAErryXVgUOfv2YqNllqGeg==",
+      "dev": true,
+      "license": "MIT"
+    },
+    "node_modules/tinyexec": {
+      "version": "0.3.2",
+      "resolved": "https://registry.npmjs.org/tinyexec/-/tinyexec-0.3.2.tgz",
+      "integrity": "sha512-KQQR9yN7R5+OSwaK0XQoj22pwHoTlgYqmUscPYoknOoWCWfj/5/ABTMRi69FrKU5ffPVh5QcFikpWJI/P1ocHA==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/tinyglobby": {
       "version": "0.2.16",
       "resolved": "https://registry.npmjs.org/tinyglobby/-/tinyglobby-0.2.16.tgz",
@@ -6655,6 +8117,56 @@
         "url": "https://github.com/sponsors/jonschlinkert"
       }
     },
+    "node_modules/tinypool": {
+      "version": "1.1.1",
+      "resolved": "https://registry.npmjs.org/tinypool/-/tinypool-1.1.1.tgz",
+      "integrity": "sha512-Zba82s87IFq9A9XmjiX5uZA/ARWDrB03OHlq+Vw1fSdt0I+4/Kutwy8BP4Y/y/aORMo61FQ0vIb5j44vSo5Pkg==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": "^18.0.0 || >=20.0.0"
+      }
+    },
+    "node_modules/tinyrainbow": {
+      "version": "2.0.0",
+      "resolved": "https://registry.npmjs.org/tinyrainbow/-/tinyrainbow-2.0.0.tgz",
+      "integrity": "sha512-op4nsTR47R6p0vMUUoYl/a+ljLFVtlfaXkLQmqfLR1qHma1h/ysYk4hEXZ880bf2CYgTskvTa/e196Vd5dDQXw==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=14.0.0"
+      }
+    },
+    "node_modules/tinyspy": {
+      "version": "4.0.4",
+      "resolved": "https://registry.npmjs.org/tinyspy/-/tinyspy-4.0.4.tgz",
+      "integrity": "sha512-azl+t0z7pw/z958Gy9svOTuzqIk6xq+NSheJzn5MMWtWTFywIacg2wUlzKFGtt3cthx0r2SxMK0yzJOR0IES7Q==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=14.0.0"
+      }
+    },
+    "node_modules/tldts": {
+      "version": "7.4.10",
+      "resolved": "https://registry.npmjs.org/tldts/-/tldts-7.4.10.tgz",
+      "integrity": "sha512-GgouD1B+sWwvkaEq8vXC15DjQitxbvs12oIXELpconwm+Tg3zfcEv4jgzq3vtKverDXsg3VI8aRgNL2Nra0Iog==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "tldts-core": "^7.4.10"
+      },
+      "bin": {
+        "tldts": "bin/cli.js"
+      }
+    },
+    "node_modules/tldts-core": {
+      "version": "7.4.10",
+      "resolved": "https://registry.npmjs.org/tldts-core/-/tldts-core-7.4.10.tgz",
+      "integrity": "sha512-KnQjp53ZekKgm/r3l+u8kJGGzYgrWdP8+Mql7a4vijh2WE0IrZWspQj/TpTxDho/YxO+AnOZnIjQcCD+q6iJsw==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/to-regex-range": {
       "version": "5.0.1",
       "resolved": "https://registry.npmjs.org/to-regex-range/-/to-regex-range-5.0.1.tgz",
@@ -6668,6 +8180,32 @@
         "node": ">=8.0"
       }
     },
+    "node_modules/tough-cookie": {
+      "version": "6.0.2",
+      "resolved": "https://registry.npmjs.org/tough-cookie/-/tough-cookie-6.0.2.tgz",
+      "integrity": "sha512-exgYmnmL/sJpR3upZfXG5PoatXQii55xAiXGXzY+sROLZ/Y+SLcp9PgJNI9Vz37HpQ74WvDcLT8eqm+kV3FzrA==",
+      "dev": true,
+      "license": "BSD-3-Clause",
+      "dependencies": {
+        "tldts": "^7.0.5"
+      },
+      "engines": {
+        "node": ">=16"
+      }
+    },
+    "node_modules/tr46": {
+      "version": "6.0.0",
+      "resolved": "https://registry.npmjs.org/tr46/-/tr46-6.0.0.tgz",
+      "integrity": "sha512-bLVMLPtstlZ4iMQHpFHTR7GAGj2jxi8Dg0s2h2MafAE4uSWF98FC/3MomU51iQAMf8/qDUbKWf5GxuvvVcXEhw==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "punycode": "^2.3.1"
+      },
+      "engines": {
+        "node": ">=20"
+      }
+    },
     "node_modules/ts-api-utils": {
       "version": "2.5.0",
       "resolved": "https://registry.npmjs.org/ts-api-utils/-/ts-api-utils-2.5.0.tgz",
@@ -6963,6 +8501,268 @@
         "punycode": "^2.1.0"
       }
     },
+    "node_modules/vite": {
+      "version": "7.3.6",
+      "resolved": "https://registry.npmjs.org/vite/-/vite-7.3.6.tgz",
+      "integrity": "sha512-4XP60spRGjSZFf1qYH+dJIkK2znL3zQfl9KkOV9MkkRR/3Dls0dxaBsQPTloEc5BLXWPL9vsOxopxyKoMmDueg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "esbuild": "^0.27.0 || ^0.28.0",
+        "fdir": "^6.5.0",
+        "picomatch": "^4.0.3",
+        "postcss": "^8.5.6",
+        "rollup": "^4.43.0",
+        "tinyglobby": "^0.2.15"
+      },
+      "bin": {
+        "vite": "bin/vite.js"
+      },
+      "engines": {
+        "node": "^20.19.0 || >=22.12.0"
+      },
+      "funding": {
+        "url": "https://github.com/vitejs/vite?sponsor=1"
+      },
+      "optionalDependencies": {
+        "fsevents": "~2.3.3"
+      },
+      "peerDependencies": {
+        "@types/node": "^20.19.0 || >=22.12.0",
+        "jiti": ">=1.21.0",
+        "less": "^4.0.0",
+        "lightningcss": "^1.21.0",
+        "sass": "^1.70.0",
+        "sass-embedded": "^1.70.0",
+        "stylus": ">=0.54.8",
+        "sugarss": "^5.0.0",
+        "terser": "^5.16.0",
+        "tsx": "^4.8.1",
+        "yaml": "^2.4.2"
+      },
+      "peerDependenciesMeta": {
+        "@types/node": {
+          "optional": true
+        },
+        "jiti": {
+          "optional": true
+        },
+        "less": {
+          "optional": true
+        },
+        "lightningcss": {
+          "optional": true
+        },
+        "sass": {
+          "optional": true
+        },
+        "sass-embedded": {
+          "optional": true
+        },
+        "stylus": {
+          "optional": true
+        },
+        "sugarss": {
+          "optional": true
+        },
+        "terser": {
+          "optional": true
+        },
+        "tsx": {
+          "optional": true
+        },
+        "yaml": {
+          "optional": true
+        }
+      }
+    },
+    "node_modules/vite-node": {
+      "version": "3.2.4",
+      "resolved": "https://registry.npmjs.org/vite-node/-/vite-node-3.2.4.tgz",
+      "integrity": "sha512-EbKSKh+bh1E1IFxeO0pg1n4dvoOTt0UDiXMd/qn++r98+jPO1xtJilvXldeuQ8giIB5IkpjCgMleHMNEsGH6pg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "cac": "^6.7.14",
+        "debug": "^4.4.1",
+        "es-module-lexer": "^1.7.0",
+        "pathe": "^2.0.3",
+        "vite": "^5.0.0 || ^6.0.0 || ^7.0.0-0"
+      },
+      "bin": {
+        "vite-node": "vite-node.mjs"
+      },
+      "engines": {
+        "node": "^18.0.0 || ^20.0.0 || >=22.0.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/vitest"
+      }
+    },
+    "node_modules/vite/node_modules/fdir": {
+      "version": "6.5.0",
+      "resolved": "https://registry.npmjs.org/fdir/-/fdir-6.5.0.tgz",
+      "integrity": "sha512-tIbYtZbucOs0BRGqPJkshJUYdL+SDH7dVM8gjy+ERp3WAUjLEFJE+02kanyHtwjWOnwrKYBiwAmM0p4kLJAnXg==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=12.0.0"
+      },
+      "peerDependencies": {
+        "picomatch": "^3 || ^4"
+      },
+      "peerDependenciesMeta": {
+        "picomatch": {
+          "optional": true
+        }
+      }
+    },
+    "node_modules/vite/node_modules/picomatch": {
+      "version": "4.0.5",
+      "resolved": "https://registry.npmjs.org/picomatch/-/picomatch-4.0.5.tgz",
+      "integrity": "sha512-RvwwcruNjI1ncT5xRakeyS9Lf8lcItv34KD+aif+VH9kduAyfYBipGh12274xtenIPZ119/R9BdTBa8gAwSh0A==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=12"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/jonschlinkert"
+      }
+    },
+    "node_modules/vitest": {
+      "version": "3.2.7",
+      "resolved": "https://registry.npmjs.org/vitest/-/vitest-3.2.7.tgz",
+      "integrity": "sha512-KrxIJ62Fd89gfysR4WotlgZABiz2dqFPgqGzX7s+CwsqLFomRH7777ZcrOD6+WVAh7khPQP41A+BKbpcJFrdEg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@types/chai": "^5.2.2",
+        "@vitest/expect": "3.2.7",
+        "@vitest/mocker": "3.2.7",
+        "@vitest/pretty-format": "^3.2.7",
+        "@vitest/runner": "3.2.7",
+        "@vitest/snapshot": "3.2.7",
+        "@vitest/spy": "3.2.7",
+        "@vitest/utils": "3.2.7",
+        "chai": "^5.2.0",
+        "debug": "^4.4.1",
+        "expect-type": "^1.2.1",
+        "magic-string": "^0.30.17",
+        "pathe": "^2.0.3",
+        "picomatch": "^4.0.2",
+        "std-env": "^3.9.0",
+        "tinybench": "^2.9.0",
+        "tinyexec": "^0.3.2",
+        "tinyglobby": "^0.2.14",
+        "tinypool": "^1.1.1",
+        "tinyrainbow": "^2.0.0",
+        "vite": "^5.0.0 || ^6.0.0 || ^7.0.0-0",
+        "vite-node": "3.2.4",
+        "why-is-node-running": "^2.3.0"
+      },
+      "bin": {
+        "vitest": "vitest.mjs"
+      },
+      "engines": {
+        "node": "^18.0.0 || ^20.0.0 || >=22.0.0"
+      },
+      "funding": {
+        "url": "https://opencollective.com/vitest"
+      },
+      "peerDependencies": {
+        "@edge-runtime/vm": "*",
+        "@types/debug": "^4.1.12",
+        "@types/node": "^18.0.0 || ^20.0.0 || >=22.0.0",
+        "@vitest/browser": "3.2.7",
+        "@vitest/ui": "3.2.7",
+        "happy-dom": "*",
+        "jsdom": "*"
+      },
+      "peerDependenciesMeta": {
+        "@edge-runtime/vm": {
+          "optional": true
+        },
+        "@types/debug": {
+          "optional": true
+        },
+        "@types/node": {
+          "optional": true
+        },
+        "@vitest/browser": {
+          "optional": true
+        },
+        "@vitest/ui": {
+          "optional": true
+        },
+        "happy-dom": {
+          "optional": true
+        },
+        "jsdom": {
+          "optional": true
+        }
+      }
+    },
+    "node_modules/vitest/node_modules/picomatch": {
+      "version": "4.0.5",
+      "resolved": "https://registry.npmjs.org/picomatch/-/picomatch-4.0.5.tgz",
+      "integrity": "sha512-RvwwcruNjI1ncT5xRakeyS9Lf8lcItv34KD+aif+VH9kduAyfYBipGh12274xtenIPZ119/R9BdTBa8gAwSh0A==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=12"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/jonschlinkert"
+      }
+    },
+    "node_modules/w3c-xmlserializer": {
+      "version": "5.0.0",
+      "resolved": "https://registry.npmjs.org/w3c-xmlserializer/-/w3c-xmlserializer-5.0.0.tgz",
+      "integrity": "sha512-o8qghlI8NZHU1lLPrpi2+Uq7abh4GGPpYANlalzWxyWteJOCsr/P+oPBA49TOLu5FTZO4d3F9MnWJfiMo4BkmA==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "xml-name-validator": "^5.0.0"
+      },
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/webidl-conversions": {
+      "version": "8.0.1",
+      "resolved": "https://registry.npmjs.org/webidl-conversions/-/webidl-conversions-8.0.1.tgz",
+      "integrity": "sha512-BMhLD/Sw+GbJC21C/UgyaZX41nPt8bUTg+jWyDeg7e7YN4xOM05YPSIXceACnXVtqyEw/LMClUQMtMZ+PGGpqQ==",
+      "dev": true,
+      "license": "BSD-2-Clause",
+      "engines": {
+        "node": ">=20"
+      }
+    },
+    "node_modules/whatwg-mimetype": {
+      "version": "4.0.0",
+      "resolved": "https://registry.npmjs.org/whatwg-mimetype/-/whatwg-mimetype-4.0.0.tgz",
+      "integrity": "sha512-QaKxh0eNIi2mE9p2vEdzfagOKHCcj1pJ56EEHGQOVxp8r9/iszLUUV7v89x9O1p/T+NlTM5W7jW6+cz4Fq1YVg==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/whatwg-url": {
+      "version": "15.1.0",
+      "resolved": "https://registry.npmjs.org/whatwg-url/-/whatwg-url-15.1.0.tgz",
+      "integrity": "sha512-2ytDk0kiEj/yu90JOAp44PVPUkO9+jVhyf+SybKlRHSDlvOOZhdPIrr7xTH64l4WixO2cP+wQIcgujkGBPPz6g==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "tr46": "^6.0.0",
+        "webidl-conversions": "^8.0.0"
+      },
+      "engines": {
+        "node": ">=20"
+      }
+    },
     "node_modules/which": {
       "version": "2.0.2",
       "resolved": "https://registry.npmjs.org/which/-/which-2.0.2.tgz",
@@ -7068,6 +8868,23 @@
         "url": "https://github.com/sponsors/ljharb"
       }
     },
+    "node_modules/why-is-node-running": {
+      "version": "2.3.0",
+      "resolved": "https://registry.npmjs.org/why-is-node-running/-/why-is-node-running-2.3.0.tgz",
+      "integrity": "sha512-hUrmaWBdVDcxvYqnyh09zunKzROWjbZTiNy8dBEjkS7ehEDQibXJ7XvlmtbwuTclUiIyN+CyXQD4Vmko8fNm8w==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "siginfo": "^2.0.0",
+        "stackback": "0.0.2"
+      },
+      "bin": {
+        "why-is-node-running": "cli.js"
+      },
+      "engines": {
+        "node": ">=8"
+      }
+    },
     "node_modules/word-wrap": {
       "version": "1.2.5",
       "resolved": "https://registry.npmjs.org/word-wrap/-/word-wrap-1.2.5.tgz",
@@ -7078,6 +8895,45 @@
         "node": ">=0.10.0"
       }
     },
+    "node_modules/ws": {
+      "version": "8.21.3",
+      "resolved": "https://registry.npmjs.org/ws/-/ws-8.21.3.tgz",
+      "integrity": "sha512-201TZ/kPWxoPr/OKWjquZR1SWKXcvxdH+e1xrx89b3YbmzLMFCLfnaG1HFIgWzJOEWZ7MvpK++odZufgYR50Rw==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=10.0.0"
+      },
+      "peerDependencies": {
+        "bufferutil": "^4.0.1",
+        "utf-8-validate": ">=5.0.2"
+      },
+      "peerDependenciesMeta": {
+        "bufferutil": {
+          "optional": true
+        },
+        "utf-8-validate": {
+          "optional": true
+        }
+      }
+    },
+    "node_modules/xml-name-validator": {
+      "version": "5.0.0",
+      "resolved": "https://registry.npmjs.org/xml-name-validator/-/xml-name-validator-5.0.0.tgz",
+      "integrity": "sha512-EvGK8EJ3DhaHfbRlETOWAS5pO9MZITeauHKJyb8wyajUfQUenkIg2MvLDTZ4T/TgIcm3HU0TFBgWWboAZ30UHg==",
+      "dev": true,
+      "license": "Apache-2.0",
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/xmlchars": {
+      "version": "2.2.0",
+      "resolved": "https://registry.npmjs.org/xmlchars/-/xmlchars-2.2.0.tgz",
+      "integrity": "sha512-JZnDKK8B0RCDw84FNdDAIpZK+JuJw+s7Lz8nksI7SIuU3UXJJslUthsi+uWBUYOwPFwW7W7PRLRfUKpxjtjFCw==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/yallist": {
       "version": "3.1.1",
       "resolved": "https://registry.npmjs.org/yallist/-/yallist-3.1.1.tgz",
diff --git a/package.json b/package.json
index 34d8cee..55dabbf 100644
--- a/package.json
+++ b/package.json
@@ -9,7 +9,9 @@
     "start": "next start -p 3100",
     "lint": "eslint .",
     "typecheck": "tsc --noEmit",
-    "content:check": "node --import tsx scripts/validate-content.ts"
+    "content:check": "node --import tsx scripts/validate-content.ts",
+    "test": "vitest run",
+    "test:watch": "vitest"
   },
   "dependencies": {
     "next": "16.2.4",
@@ -20,13 +22,17 @@
   },
   "devDependencies": {
     "@tailwindcss/postcss": "^4",
+    "@testing-library/jest-dom": "^6.9.1",
+    "@testing-library/react": "^16.3.2",
     "@types/node": "^20",
     "@types/react": "^19",
     "@types/react-dom": "^19",
     "eslint": "^9",
     "eslint-config-next": "16.2.4",
+    "jsdom": "^27.0.1",
     "tailwindcss": "^4",
     "tsx": "^4.23.0",
-    "typescript": "^5"
+    "typescript": "^5",
+    "vitest": "^3.2.4"
   }
 }
diff --git a/src/lib/portfolio.test.ts b/src/lib/portfolio.test.ts
new file mode 100644
index 0000000..081ed22
--- /dev/null
+++ b/src/lib/portfolio.test.ts
@@ -0,0 +1,379 @@
+import { resolve } from "node:path";
+
+import contactJson from "@/content/contact.json";
+import linksJson from "@/content/links.json";
+import presentationJson from "@/content/presentation.json";
+import projectsJson from "@/content/projects.json";
+import resumeJson from "@/content/resume.json";
+import siteJson from "@/content/site.json";
+import { SITE_DESIGN_IDS } from "@/designs/config";
+import { describe, expect, it } from "vitest";
+
+import { validatePortfolioAssets } from "./content-assets";
+import { loadPortfolioSource, PortfolioContentError } from "./content-loader";
+import {
+  getFeaturedProjects,
+  getPortfolioContent,
+  getProjectCardLinks,
+  getProjectMetricValue,
+  getResumeProjects,
+  getTemplateHref,
+  isProjectLive,
+  resolveContentDebug,
+  resolveHomeTemplateId,
+  type PortfolioProject,
+  type ProjectMetricFilter,
+  type SiteDesignId,
+} from "./portfolio";
+
+const designIds = [
+  "design",
+  "classic",
+  "editorial",
+  "brutalist",
+  "cinematic",
+] as const satisfies readonly SiteDesignId[];
+
+function projectMatchesFilter(
+  project: PortfolioProject,
+  filter: ProjectMetricFilter | undefined,
+) {
+  if (!filter) return true;
+
+  return (
+    (!filter.projectIds || filter.projectIds.includes(project.id)) &&
+    (!filter.groupIds || filter.groupIds.includes(project.groupId)) &&
+    (!filter.tags || filter.tags.every((tag) => project.tags.includes(tag))) &&
+    (filter.featured === undefined ||
+      Boolean(project.featured) === filter.featured) &&
+    (!filter.deploymentStatuses ||
+      filter.deploymentStatuses.includes(project.deployment.status))
+  );
+}
+
+function captureContentError(run: () => unknown) {
+  let caught: unknown;
+
+  try {
+    run();
+  } catch (error) {
+    caught = error;
+  }
+
+  expect(caught).toBeInstanceOf(PortfolioContentError);
+  return caught as PortfolioContentError;
+}
+
+describe("portfolio content", () => {
+  it("loads and derives the reusable projects content model", () => {
+    const source = validatePortfolioAssets(
+      loadPortfolioSource(),
+      resolve(process.cwd(), "public"),
+    );
+    const content = getPortfolioContent();
+    const sourceGroups = new Map(
+      source.projects.groups.map((group) => [group.id, group]),
+    );
+
+    expect(source.projects.groups.length).toBeGreaterThan(0);
+    expect(source.projects.metrics.length).toBeGreaterThan(0);
+    expect(source.projects.items.length).toBeGreaterThan(0);
+    expect(new Set(source.projects.groups.map((group) => group.id)).size).toBe(
+      source.projects.groups.length,
+    );
+    expect(new Set(source.projects.metrics.map((metric) => metric.id)).size).toBe(
+      source.projects.metrics.length,
+    );
+    expect(new Set(source.projects.items.map((project) => project.id)).size).toBe(
+      source.projects.items.length,
+    );
+
+    expect(content.projectGroups.map((group) => group.order)).toEqual(
+      [...content.projectGroups.map((group) => group.order)].sort(
+        (left, right) => left - right,
+      ),
+    );
+    for (const project of content.projects) {
+      expect(project.category).toBe(sourceGroups.get(project.groupId)?.label);
+      expect(project.tags.length).toBeGreaterThan(0);
+    }
+  });
+
+  it("exposes the complete five-design contract with editorial as the default", () => {
+    const { presentation } = getPortfolioContent();
+
+    expect(presentation.defaultHomeTemplate).toBe("editorial");
+    expect(presentation.templates.map((template) => template.id)).toEqual(
+      designIds,
+    );
+    expect(SITE_DESIGN_IDS).toEqual(designIds);
+
+    const configurableSectionOrders = [
+      presentation.home.editorial.sections,
+      presentation.home.brutalist.sections,
+      presentation.home.cinematic.sections,
+    ];
+
+    for (const sections of configurableSectionOrders) {
+      expect(sections.length).toBeGreaterThan(0);
+      expect(new Set(sections).size).toBe(sections.length);
+    }
+
+    expect(presentation.ui.skipLinkLabel).toMatch(/\S/);
+    expect(presentation.ui.emptyStates.projectDetails).toMatch(/\S/);
+  });
+
+  it("keeps mutable assets in the documented content boundaries", () => {
+    const content = getPortfolioContent();
+    const assetPaths = [
+      content.profile.photo?.src,
+      content.resume.downloadUrl,
+      ...content.projects.flatMap((project) => [
+        project.screenshot.src,
+        ...project.screenshots.map((image) => image.src),
+      ]),
+    ].filter((path): path is string => Boolean(path));
+
+    expect(assetPaths.length).toBeGreaterThan(0);
+    for (const path of assetPaths) {
+      expect(path).toMatch(/^\/(?:content|template)\//);
+    }
+
+    expect(
+      getPortfolioContent({
+        NEXT_PUBLIC_DASHBOARD_URL: "https://environment-specific.example",
+      }),
+    ).toEqual(content);
+  });
+
+  it("reports missing assets with the source file and JSON path", () => {
+    let caught: unknown;
+
+    try {
+      validatePortfolioAssets(
+        loadPortfolioSource(),
+        resolve(process.cwd(), ".missing-public-root"),
+      );
+    } catch (error) {
+      caught = error;
+    }
+
+    expect(caught).toBeInstanceOf(PortfolioContentError);
+    const contentError = caught as PortfolioContentError;
+    expect(contentError.issues.length).toBeGreaterThan(0);
+    expect(
+      contentError.issues.some(
+        (issue) =>
+          issue.file.startsWith("src/content/") &&
+          issue.path.startsWith("$.") &&
+          issue.message.includes("does not exist under public/"),
+      ),
+    ).toBe(true);
+    expect(contentError.message).toContain("Portfolio content validation failed");
+  });
+
+  it("rejects duplicate IDs, missing designs, and unsupported navigation", () => {
+    const projects = structuredClone(projectsJson);
+    projects.items.push(structuredClone(projects.items[0]));
+    const presentation = structuredClone(presentationJson);
+    presentation.templates = presentation.templates.slice(0, -1);
+    const site = structuredClone(siteJson);
+    site.navigation.push({ label: "Unknown", href: "/not-a-route" });
+
+    const error = captureContentError(() =>
+      loadPortfolioSource({ projects, presentation, site }),
+    );
+
+    expect(error.issues).toEqual(
+      expect.arrayContaining([
+        expect.objectContaining({
+          file: "src/content/projects.json",
+          message: expect.stringContaining("Duplicate project id"),
+        }),
+        expect.objectContaining({
+          file: "src/content/presentation.json",
+          message: expect.stringContaining("Missing supported site design"),
+        }),
+        expect.objectContaining({
+          file: "src/content/site.json",
+          message: expect.stringContaining("Unsupported internal navigation route"),
+        }),
+      ]),
+    );
+  });
+
+  it("rejects navigation and references to disabled content", () => {
+    const projects = structuredClone(projectsJson);
+    const disabledProjectId = projects.items[0].id;
+    projects.items[0].enabled = false;
+    const resume = structuredClone(resumeJson);
+    resume.projectIds = [disabledProjectId];
+
+    const links = structuredClone(linksJson);
+    const preferredLink = links.find((link) => link.id !== undefined);
+    expect(preferredLink?.id).toBeDefined();
+    if (!preferredLink?.id) return;
+    preferredLink.enabled = false;
+    const contact = structuredClone(contactJson);
+    contact.preferred = [preferredLink.id];
+
+    const site = structuredClone(siteJson);
+    site.pages.projects = false;
+
+    const error = captureContentError(() =>
+      loadPortfolioSource({ contact, links, projects, resume, site }),
+    );
+
+    expect(error.issues).toEqual(
+      expect.arrayContaining([
+        expect.objectContaining({
+          path: expect.stringContaining("navigation"),
+          message: expect.stringContaining("disabled page"),
+        }),
+        expect.objectContaining({
+          file: "src/content/resume.json",
+          message: expect.stringContaining(disabledProjectId),
+        }),
+        expect.objectContaining({
+          file: "src/content/contact.json",
+          message: expect.stringContaining(preferredLink.id),
+        }),
+      ]),
+    );
+  });
+
+  it("rejects unsupported internal links and missing project routes", () => {
+    const links = structuredClone(linksJson);
+    links[0].href = "/not-a-route";
+    links[0].external = false;
+
+    const projects = structuredClone(projectsJson);
+    const projectWithLinks = projects.items.find(
+      (project) => project.links.length > 0,
+    );
+    expect(projectWithLinks).toBeDefined();
+    if (!projectWithLinks) return;
+    projectWithLinks.links[0].href = "/projects/not-a-project";
+    projectWithLinks.links[0].external = false;
+
+    const error = captureContentError(() =>
+      loadPortfolioSource({ links, projects }),
+    );
+
+    expect(error.issues).toEqual(
+      expect.arrayContaining([
+        expect.objectContaining({
+          file: "src/content/links.json",
+          message: expect.stringContaining("Unsupported internal link route"),
+        }),
+        expect.objectContaining({
+          file: "src/content/projects.json",
+          message: expect.stringContaining("unknown or disabled project"),
+        }),
+      ]),
+    );
+  });
+
+  it("computes every declared metric from generic filters", () => {
+    const content = getPortfolioContent();
+
+    for (const metric of content.projectMetrics) {
+      const matchingProjects = content.projects.filter((project) =>
+        projectMatchesFilter(project, metric.filter),
+      );
+      const expectedValue =
+        metric.aggregate === "highlights"
+          ? matchingProjects.reduce(
+              (total, project) => total + project.highlights.length,
+              0,
+            )
+          : matchingProjects.length;
+
+      expect(getProjectMetricValue(metric.id, content)).toBe(expectedValue);
+    }
+
+    expect(getProjectMetricValue("not-a-declared-metric", content)).toBe(0);
+  });
+
+  it("selects featured, resume, and card links from content flags", () => {
+    const content = getPortfolioContent();
+    const expectedResumeIds = content.resume.projectIds.filter((projectId) =>
+      content.projects.some((project) => project.id === projectId),
+    );
+
+    expect(getFeaturedProjects(content)).toEqual(
+      content.projects.filter((project) => project.featured),
+    );
+    expect(getResumeProjects(content).map((project) => project.id)).toEqual(
+      expectedResumeIds,
+    );
+
+    for (const project of content.projects) {
+      const hasEnabledDemo = project.links.some(
+        (link) => link.type === "demo" && link.enabled !== false,
+      );
+      expect(isProjectLive(project)).toBe(
+        project.deployment.status === "live" && hasEnabledDemo,
+      );
+      expect(getProjectCardLinks(project).every((link) =>
+        link.placements?.includes("card"),
+      )).toBe(true);
+      expect(
+        getProjectCardLinks(project).some((link) => link.type === "demo"),
+      ).toBe(isProjectLive(project) && project.links.some(
+        (link) => link.type === "demo" && link.placements?.includes("card"),
+      ));
+    }
+  });
+
+  it("keeps journey entries chronological without assuming owner copy", () => {
+    const { journey } = getPortfolioContent();
+    const dates = journey.map((item) => item.date);
+
+    expect(dates).toEqual([...dates].sort());
+  });
+
+  it("resolves all supported designs and falls back to editorial", () => {
+    const { presentation } = getPortfolioContent();
+
+    for (const designId of designIds) {
+      expect(resolveHomeTemplateId(designId, presentation)).toBe(designId);
+      expect(resolveHomeTemplateId([designId], presentation)).toBe(designId);
+    }
+
+    expect(resolveHomeTemplateId("missing", presentation)).toBe("editorial");
+    expect(resolveHomeTemplateId(undefined, presentation)).toBe("editorial");
+    expect(resolveContentDebug("content")).toBe(true);
+    expect(resolveContentDebug(["content"])).toBe(true);
+    expect(resolveContentDebug("off")).toBe(false);
+  });
+
+  it("propagates designs and debug state on internal links only", () => {
+    for (const designId of designIds) {
+      expect(getTemplateHref("/projects", designId)).toBe(
+        designId === "editorial"
+          ? "/projects"
+          : `/projects?view=${designId}`,
+      );
+    }
+
+    expect(getTemplateHref("/projects?page=2#featured", "cinematic")).toBe(
+      "/projects?page=2&view=cinematic#featured",
+    );
+    expect(getTemplateHref("/projects?view=classic", "editorial")).toBe(
+      "/projects",
+    );
+    expect(
+      getTemplateHref("/", "editorial", { alwaysInclude: true }),
+    ).toBe("/?view=editorial");
+    expect(
+      getTemplateHref("/projects", "brutalist", { contentDebug: true }),
+    ).toBe("/projects?view=brutalist&debug=content");
+    expect(getTemplateHref("https://example.com", "classic")).toBe(
+      "https://example.com",
+    );
+    expect(getTemplateHref("//example.com/project", "classic")).toBe(
+      "//example.com/project",
+    );
+  });
+});
diff --git a/src/test/setup.ts b/src/test/setup.ts
new file mode 100644
index 0000000..f149f27
--- /dev/null
+++ b/src/test/setup.ts
@@ -0,0 +1 @@
+import "@testing-library/jest-dom/vitest";
diff --git a/vitest.config.ts b/vitest.config.ts
new file mode 100644
index 0000000..3eabb55
--- /dev/null
+++ b/vitest.config.ts
@@ -0,0 +1,16 @@
+import { defineConfig } from "vitest/config";
+import { fileURLToPath } from "node:url";
+
+export default defineConfig({
+  resolve: {
+    alias: {
+      "@": fileURLToPath(new URL("./src", import.meta.url)),
+    },
+  },
+  test: {
+    environment: "jsdom",
+    globals: true,
+    include: ["src/**/*.test.{ts,tsx}"],
+    setupFiles: ["src/test/setup.ts"],
+  },
+});
