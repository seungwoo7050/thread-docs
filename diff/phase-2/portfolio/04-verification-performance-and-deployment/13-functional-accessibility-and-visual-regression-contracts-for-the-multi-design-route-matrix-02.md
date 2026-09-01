## `test(e2e): 다섯 디자인의 route matrix 검증`

diff --git a/.gitignore b/.gitignore
index 0c1a76f..ef541f3 100644
--- a/.gitignore
+++ b/.gitignore
@@ -1,3 +1,5 @@
+# See https://help.github.com/articles/ignoring-files/ for more about ignoring files.
+
 # dependencies
 /node_modules
 /.pnp
@@ -13,22 +15,31 @@
 /playwright-report
 /test-results
 /output
+/.playwright-cli
 
-# Next.js
+# next.js
 /.next/
 /out/
 
 # production
 /build
 
-# local files
+# misc
 .DS_Store
 *.pem
+
+# debug
 npm-debug.log*
 yarn-debug.log*
 yarn-error.log*
 .pnpm-debug.log*
+
+# env files (can opt-in for committing if needed)
 .env*
+
+# vercel
 .vercel
+
+# typescript
 *.tsbuildinfo
 next-env.d.ts
diff --git a/package-lock.json b/package-lock.json
index e6a6d8d..d8078b1 100644
--- a/package-lock.json
+++ b/package-lock.json
@@ -15,6 +15,7 @@
         "zod": "^4.4.3"
       },
       "devDependencies": {
+        "@playwright/test": "^1.59.1",
         "@tailwindcss/postcss": "^4",
         "@testing-library/jest-dom": "^6.9.1",
         "@testing-library/react": "^16.3.2",
@@ -30,17 +31,10 @@
         "vitest": "^3.2.4"
       }
     },
-    "node_modules/@acemir/cssom": {
-      "version": "0.9.31",
-      "resolved": "https://registry.npmjs.org/@acemir/cssom/-/cssom-0.9.31.tgz",
-      "integrity": "sha512-ZnR3GSaH+/vJ0YlHau21FjfLYjMpYVIzTD8M8vIEQvIGxeOXyXdzCI140rrCY862p/C/BbzWsjc1dgnM9mkoTA==",
-      "dev": true,
-      "license": "MIT"
-    },
     "node_modules/@adobe/css-tools": {
-      "version": "4.5.0",
-      "resolved": "https://registry.npmjs.org/@adobe/css-tools/-/css-tools-4.5.0.tgz",
-      "integrity": "sha512-6OzddxPio9UiWTCemp4N8cYLV2ZN1ncRnV1cVGtve7dhPOtRkleRyx32GQCYSwDYgaHU3USMm84tNsvKzRCa1Q==",
+      "version": "4.4.4",
+      "resolved": "https://registry.npmjs.org/@adobe/css-tools/-/css-tools-4.4.4.tgz",
+      "integrity": "sha512-Elp+iwUx5rN5+Y8xLt5/GRoG20WGoDCQ/1Fb+1LiGtvwbDavuSk0jhD/eZdckHAuzcDzccnkv+rEjyWfRx18gg==",
       "dev": true,
       "license": "MIT"
     },
@@ -72,9 +66,9 @@
       }
     },
     "node_modules/@asamuzakjp/css-color/node_modules/lru-cache": {
-      "version": "11.5.2",
-      "resolved": "https://registry.npmjs.org/lru-cache/-/lru-cache-11.5.2.tgz",
-      "integrity": "sha512-4pfM1Ff0x50o0tQwb5ucw/RzNyD0/YJME6IVcStalZuMWxdt3sR3huStTtxz4PUmvZfRguvDejasvQ2kifR11g==",
+      "version": "11.3.6",
+      "resolved": "https://registry.npmjs.org/lru-cache/-/lru-cache-11.3.6.tgz",
+      "integrity": "sha512-Gf/KoL3C/MlI7Bt0PGI9I+TeTC/I6r/csU58N4BSNc4lppLBeKsOdFYkK+dX0ABDUMJNfCHTyPpzwwO21Awd3A==",
       "dev": true,
       "license": "BlueOak-1.0.0",
       "engines": {
@@ -96,9 +90,9 @@
       }
     },
     "node_modules/@asamuzakjp/dom-selector/node_modules/lru-cache": {
-      "version": "11.5.2",
-      "resolved": "https://registry.npmjs.org/lru-cache/-/lru-cache-11.5.2.tgz",
-      "integrity": "sha512-4pfM1Ff0x50o0tQwb5ucw/RzNyD0/YJME6IVcStalZuMWxdt3sR3huStTtxz4PUmvZfRguvDejasvQ2kifR11g==",
+      "version": "11.3.6",
+      "resolved": "https://registry.npmjs.org/lru-cache/-/lru-cache-11.3.6.tgz",
+      "integrity": "sha512-Gf/KoL3C/MlI7Bt0PGI9I+TeTC/I6r/csU58N4BSNc4lppLBeKsOdFYkK+dX0ABDUMJNfCHTyPpzwwO21Awd3A==",
       "dev": true,
       "license": "BlueOak-1.0.0",
       "engines": {
@@ -305,9 +299,9 @@
       }
     },
     "node_modules/@babel/runtime": {
-      "version": "7.29.7",
-      "resolved": "https://registry.npmjs.org/@babel/runtime/-/runtime-7.29.7.tgz",
-      "integrity": "sha512-Nq8OhGWiZIZGV6hLHoyAKLLcJihP/xFeBMGJoUrxTX2psI8dCifzLhZISFb+VWS3wFMRDmCGw5R+dOySCqPLhw==",
+      "version": "7.29.2",
+      "resolved": "https://registry.npmjs.org/@babel/runtime/-/runtime-7.29.2.tgz",
+      "integrity": "sha512-JiDShH45zKHWyGe4ZNVRrCjBz8Nh9TMmZG1kh4QTK8hCBTWBi8Da+i7s1fJw7/lYpM4ccepSNfqzZ/QvABBi5g==",
       "dev": true,
       "license": "MIT",
       "engines": {
@@ -363,9 +357,9 @@
       }
     },
     "node_modules/@csstools/color-helpers": {
-      "version": "6.1.0",
-      "resolved": "https://registry.npmjs.org/@csstools/color-helpers/-/color-helpers-6.1.0.tgz",
-      "integrity": "sha512-064IFJdjTfUqnjpCVpMOdbr8FLQBhinbZj6yRv2An2E41O/pLEXqfFRWqGq/SxlE5PEUYTlvWsG2r8MswAVvkg==",
+      "version": "6.0.2",
+      "resolved": "https://registry.npmjs.org/@csstools/color-helpers/-/color-helpers-6.0.2.tgz",
+      "integrity": "sha512-LMGQLS9EuADloEFkcTBR3BwV/CGHV7zyDxVRtVDTwdI2Ca4it0CCVTT9wCkxSgokjE5Ho41hEPgb8OEUwoXr6Q==",
       "dev": true,
       "funding": [
         {
@@ -383,9 +377,9 @@
       }
     },
     "node_modules/@csstools/css-calc": {
-      "version": "3.3.0",
-      "resolved": "https://registry.npmjs.org/@csstools/css-calc/-/css-calc-3.3.0.tgz",
-      "integrity": "sha512-c5ihYsPkdG6JCkU2zTMm4+k6r7RXuGxtWYhu5DHMIiF1FHzrfmHL5so11AoFpUv/tu61xfcmT4AmKoFfMPoqdQ==",
+      "version": "3.2.0",
+      "resolved": "https://registry.npmjs.org/@csstools/css-calc/-/css-calc-3.2.0.tgz",
+      "integrity": "sha512-bR9e6o2BDB12jzN/gIbjHa5wLJ4UjD1CB9pM7ehlc0ddk6EBz+yYS1EV2MF55/HUxrHcB/hehAyt5vhsA3hx7w==",
       "dev": true,
       "funding": [
         {
@@ -407,9 +401,9 @@
       }
     },
     "node_modules/@csstools/css-color-parser": {
-      "version": "4.1.10",
-      "resolved": "https://registry.npmjs.org/@csstools/css-color-parser/-/css-color-parser-4.1.10.tgz",
-      "integrity": "sha512-UZhQLIUyJaaMepqehrCODwCg2KW25vFvLWBmqYFaPclYvvxzj/sG8LBOhBFCp11i9uE7t1EyS+RAoV9tztPFyw==",
+      "version": "4.1.0",
+      "resolved": "https://registry.npmjs.org/@csstools/css-color-parser/-/css-color-parser-4.1.0.tgz",
+      "integrity": "sha512-U0KhLYmy2GVj6q4T3WaAe6NPuFYCPQoE3b0dRGxejWDgcPp8TP7S5rVdM5ZrFaqu4N67X8YaPBw14dQSYx3IyQ==",
       "dev": true,
       "funding": [
         {
@@ -423,8 +417,8 @@
       ],
       "license": "MIT",
       "dependencies": {
-        "@csstools/color-helpers": "^6.1.0",
-        "@csstools/css-calc": "^3.3.0"
+        "@csstools/color-helpers": "^6.0.2",
+        "@csstools/css-calc": "^3.2.0"
       },
       "engines": {
         "node": ">=20.19.0"
@@ -458,9 +452,9 @@
       }
     },
     "node_modules/@csstools/css-syntax-patches-for-csstree": {
-      "version": "1.1.7",
-      "resolved": "https://registry.npmjs.org/@csstools/css-syntax-patches-for-csstree/-/css-syntax-patches-for-csstree-1.1.7.tgz",
-      "integrity": "sha512-fQ+05118eQS1cofO3aJpB5efgpBZMvIzwr/sbC8kDLVA5XLG8q1kJV5yzrUAI1f7lvhPnm8fgIjzFB8/O/5Dig==",
+      "version": "1.1.3",
+      "resolved": "https://registry.npmjs.org/@csstools/css-syntax-patches-for-csstree/-/css-syntax-patches-for-csstree-1.1.3.tgz",
+      "integrity": "sha512-SH60bMfrRCJF3morcdk57WklujF4Jr/EsQUzqkarfHXEFcAR1gg7fS/chAE922Sehgzc1/+Tz5H3Ypa1HiEKrg==",
       "dev": true,
       "funding": [
         {
@@ -536,9 +530,9 @@
       }
     },
     "node_modules/@esbuild/aix-ppc64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/aix-ppc64/-/aix-ppc64-0.28.2.tgz",
-      "integrity": "sha512-XExcO+dvLKvVtNTibSTBej1NCAbaGhWn9Ww1ZPx80qsahhPFe/8jgWP0IchNe0F3HwkU7n8ejhH8bjonqht8mQ==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/aix-ppc64/-/aix-ppc64-0.27.7.tgz",
+      "integrity": "sha512-EKX3Qwmhz1eMdEJokhALr0YiD0lhQNwDqkPYyPhiSwKrh7/4KRjQc04sZ8db+5DVVnZ1LmbNDI1uAMPEUBnQPg==",
       "cpu": [
         "ppc64"
       ],
@@ -553,9 +547,9 @@
       }
     },
     "node_modules/@esbuild/android-arm": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/android-arm/-/android-arm-0.28.2.tgz",
-      "integrity": "sha512-kXXoiPVVGQcnIYGOeaovwOURpniDBpSq4A03qkQ+BMQqtGG6HYap3xne9C1O1yo4TR3qxlCX5IqqmX6fFo2Lqg==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/android-arm/-/android-arm-0.27.7.tgz",
+      "integrity": "sha512-jbPXvB4Yj2yBV7HUfE2KHe4GJX51QplCN1pGbYjvsyCZbQmies29EoJbkEc+vYuU5o45AfQn37vZlyXy4YJ8RQ==",
       "cpu": [
         "arm"
       ],
@@ -570,9 +564,9 @@
       }
     },
     "node_modules/@esbuild/android-arm64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/android-arm64/-/android-arm64-0.28.2.tgz",
-      "integrity": "sha512-5YfKeeI8qWfBZIX+u2xZC3Zlb3Os/gLS2sbEKM+I4ZOcsWmHS2WLysCcQZDAFRslDUU5Oiq44gf6PYN1vGwG5A==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/android-arm64/-/android-arm64-0.27.7.tgz",
+      "integrity": "sha512-62dPZHpIXzvChfvfLJow3q5dDtiNMkwiRzPylSCfriLvZeq0a1bWChrGx/BbUbPwOrsWKMn8idSllklzBy+dgQ==",
       "cpu": [
         "arm64"
       ],
@@ -587,9 +581,9 @@
       }
     },
     "node_modules/@esbuild/android-x64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/android-x64/-/android-x64-0.28.2.tgz",
-      "integrity": "sha512-O387ite7SzUyCcy3JQX4P4bLtEA7bLLkx+esve5JHnyYfNTxcVpXZo9jhdB0lTKN44gztELTdU7nS8Nr16Fs1Q==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/android-x64/-/android-x64-0.27.7.tgz",
+      "integrity": "sha512-x5VpMODneVDb70PYV2VQOmIUUiBtY3D3mPBG8NxVk5CogneYhkR7MmM3yR/uMdITLrC1ml/NV1rj4bMJuy9MCg==",
       "cpu": [
         "x64"
       ],
@@ -604,9 +598,9 @@
       }
     },
     "node_modules/@esbuild/darwin-arm64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/darwin-arm64/-/darwin-arm64-0.28.2.tgz",
-      "integrity": "sha512-n4KqkOQrraxHJcgjM1RvwbigfQKIKJVpM7xp+KsxiyUSrRdIXnt73VhrPAx0fV44hgfmIVKjxMN9J1t5jySVkw==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/darwin-arm64/-/darwin-arm64-0.27.7.tgz",
+      "integrity": "sha512-5lckdqeuBPlKUwvoCXIgI2D9/ABmPq3Rdp7IfL70393YgaASt7tbju3Ac+ePVi3KDH6N2RqePfHnXkaDtY9fkw==",
       "cpu": [
         "arm64"
       ],
@@ -621,9 +615,9 @@
       }
     },
     "node_modules/@esbuild/darwin-x64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/darwin-x64/-/darwin-x64-0.28.2.tgz",
-      "integrity": "sha512-uq6suIWYP37qzGddBKPw5QEQPi6HiLGsO7UmkpfyaYNQ3D+rN6w6WfwH+nuqcGXWvawGwxOEroO4YGnFh95azw==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/darwin-x64/-/darwin-x64-0.27.7.tgz",
+      "integrity": "sha512-rYnXrKcXuT7Z+WL5K980jVFdvVKhCHhUwid+dDYQpH+qu+TefcomiMAJpIiC2EM3Rjtq0sO3StMV/+3w3MyyqQ==",
       "cpu": [
         "x64"
       ],
@@ -638,9 +632,9 @@
       }
     },
     "node_modules/@esbuild/freebsd-arm64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/freebsd-arm64/-/freebsd-arm64-0.28.2.tgz",
-      "integrity": "sha512-n+I0BTSRIoy+d6RPKnEVwql5UwBJolytvY4mAOIEJorKlqgPII8ix6slVVrfZ5Tnj7glIZvloylbB/EJPMWEXw==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/freebsd-arm64/-/freebsd-arm64-0.27.7.tgz",
+      "integrity": "sha512-B48PqeCsEgOtzME2GbNM2roU29AMTuOIN91dsMO30t+Ydis3z/3Ngoj5hhnsOSSwNzS+6JppqWsuhTp6E82l2w==",
       "cpu": [
         "arm64"
       ],
@@ -655,9 +649,9 @@
       }
     },
     "node_modules/@esbuild/freebsd-x64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/freebsd-x64/-/freebsd-x64-0.28.2.tgz",
-      "integrity": "sha512-78XJTJkvPs0kz2w61301PJjXl4g7q3JqiYMZ/M/yVI73EHBrCRTgkhu9oqG7vPqq+a/yadEW8aD+agKlk5xrmg==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/freebsd-x64/-/freebsd-x64-0.27.7.tgz",
+      "integrity": "sha512-jOBDK5XEjA4m5IJK3bpAQF9/Lelu/Z9ZcdhTRLf4cajlB+8VEhFFRjWgfy3M1O4rO2GQ/b2dLwCUGpiF/eATNQ==",
       "cpu": [
         "x64"
       ],
@@ -672,9 +666,9 @@
       }
     },
     "node_modules/@esbuild/linux-arm": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/linux-arm/-/linux-arm-0.28.2.tgz",
-      "integrity": "sha512-XlDnu2q5yoqems+xay6wSAcg9DDD7K9RLKZEBOMZm3ckNpJBvOX20tSfby8KfrrhINDyv9V2YVZKY/SpoGJI8w==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-arm/-/linux-arm-0.27.7.tgz",
+      "integrity": "sha512-RkT/YXYBTSULo3+af8Ib0ykH8u2MBh57o7q/DAs3lTJlyVQkgQvlrPTnjIzzRPQyavxtPtfg0EopvDyIt0j1rA==",
       "cpu": [
         "arm"
       ],
@@ -689,9 +683,9 @@
       }
     },
     "node_modules/@esbuild/linux-arm64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/linux-arm64/-/linux-arm64-0.28.2.tgz",
-      "integrity": "sha512-pW4AC0P3it8c7do9MVM4p51FzHzdM/TZrerurgRcHJ2WTa1VQ1CIq18xncfpBJw4ojkiZZrKW2yIBWBP92j6Ug==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-arm64/-/linux-arm64-0.27.7.tgz",
+      "integrity": "sha512-RZPHBoxXuNnPQO9rvjh5jdkRmVizktkT7TCDkDmQ0W2SwHInKCAV95GRuvdSvA7w4VMwfCjUiPwDi0ZO6Nfe9A==",
       "cpu": [
         "arm64"
       ],
@@ -706,9 +700,9 @@
       }
     },
     "node_modules/@esbuild/linux-ia32": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/linux-ia32/-/linux-ia32-0.28.2.tgz",
-      "integrity": "sha512-CYbnj78HsIeA+DhgUKgFCfvNsTHFhMMrinUrMZpDXJXKN8T3XViTZ/+wtHeVxEWY8ewSzTFN+nRmSwO2tZaLUQ==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-ia32/-/linux-ia32-0.27.7.tgz",
+      "integrity": "sha512-GA48aKNkyQDbd3KtkplYWT102C5sn/EZTY4XROkxONgruHPU72l+gW+FfF8tf2cFjeHaRbWpOYa/uRBz/Xq1Pg==",
       "cpu": [
         "ia32"
       ],
@@ -723,9 +717,9 @@
       }
     },
     "node_modules/@esbuild/linux-loong64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/linux-loong64/-/linux-loong64-0.28.2.tgz",
-      "integrity": "sha512-buwkd8nsph4R+ajRvw0qM5Hja/TXQow3ptzWO2EbG/cqcIkHloRrdlBtQlshyYGTNFvfkfJ5tpPLVkY4DtsPfQ==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-loong64/-/linux-loong64-0.27.7.tgz",
+      "integrity": "sha512-a4POruNM2oWsD4WKvBSEKGIiWQF8fZOAsycHOt6JBpZ+JN2n2JH9WAv56SOyu9X5IqAjqSIPTaJkqN8F7XOQ5Q==",
       "cpu": [
         "loong64"
       ],
@@ -740,9 +734,9 @@
       }
     },
     "node_modules/@esbuild/linux-mips64el": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/linux-mips64el/-/linux-mips64el-0.28.2.tgz",
-      "integrity": "sha512-ZVykbDyk7519VwiNb9Lcj9m8XM6v5V9uKPvrEMkkEedVewf+0itkhahp4HDpgERXhwLRpWFypsGbG/J8s0QjJA==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-mips64el/-/linux-mips64el-0.27.7.tgz",
+      "integrity": "sha512-KabT5I6StirGfIz0FMgl1I+R1H73Gp0ofL9A3nG3i/cYFJzKHhouBV5VWK1CSgKvVaG4q1RNpCTR2LuTVB3fIw==",
       "cpu": [
         "mips64el"
       ],
@@ -757,9 +751,9 @@
       }
     },
     "node_modules/@esbuild/linux-ppc64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/linux-ppc64/-/linux-ppc64-0.28.2.tgz",
-      "integrity": "sha512-CAXl+Dtd9UUuJd8pKKdwh6MLm3MUMiqMPmhZ3tTSXPqfyQ3vDl6R5hZdZ/kYojK4ofXtdfSv1tFq8XzWx3heNQ==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-ppc64/-/linux-ppc64-0.27.7.tgz",
+      "integrity": "sha512-gRsL4x6wsGHGRqhtI+ifpN/vpOFTQtnbsupUF5R5YTAg+y/lKelYR1hXbnBdzDjGbMYjVJLJTd2OFmMewAgwlQ==",
       "cpu": [
         "ppc64"
       ],
@@ -774,9 +768,9 @@
       }
     },
     "node_modules/@esbuild/linux-riscv64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/linux-riscv64/-/linux-riscv64-0.28.2.tgz",
-      "integrity": "sha512-GeXCej4IQtU1B+QlDV8W/RRvbzI3O/Stss+/bCXv4lZls5WGRtu2a+3JkA3i4qIUlMXpcHebWpF8AkJhATowuA==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-riscv64/-/linux-riscv64-0.27.7.tgz",
+      "integrity": "sha512-hL25LbxO1QOngGzu2U5xeXtxXcW+/GvMN3ejANqXkxZ/opySAZMrc+9LY/WyjAan41unrR3YrmtTsUpwT66InQ==",
       "cpu": [
         "riscv64"
       ],
@@ -791,9 +785,9 @@
       }
     },
     "node_modules/@esbuild/linux-s390x": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/linux-s390x/-/linux-s390x-0.28.2.tgz",
-      "integrity": "sha512-3H1weTYZPxt/WOhByszQZybS9w5lKzUn1FDMsgEChbHWQwHYQQRfBxgCcZvPhjHfKyJjIievvMmEUawJrdY9Dg==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-s390x/-/linux-s390x-0.27.7.tgz",
+      "integrity": "sha512-2k8go8Ycu1Kb46vEelhu1vqEP+UeRVj2zY1pSuPdgvbd5ykAw82Lrro28vXUrRmzEsUV0NzCf54yARIK8r0fdw==",
       "cpu": [
         "s390x"
       ],
@@ -808,9 +802,9 @@
       }
     },
     "node_modules/@esbuild/linux-x64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/linux-x64/-/linux-x64-0.28.2.tgz",
-      "integrity": "sha512-4xTZr1FUmSoQW4XIWmit3tzQrUTZM+N3P0XV8xROKYF50XfI7xeO90+1bZvNwxIufQ9hDQVRJH5YhgPVF8A/HQ==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-x64/-/linux-x64-0.27.7.tgz",
+      "integrity": "sha512-hzznmADPt+OmsYzw1EE33ccA+HPdIqiCRq7cQeL1Jlq2gb1+OyWBkMCrYGBJ+sxVzve2ZJEVeePbLM2iEIZSxA==",
       "cpu": [
         "x64"
       ],
@@ -825,9 +819,9 @@
       }
     },
     "node_modules/@esbuild/netbsd-arm64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/netbsd-arm64/-/netbsd-arm64-0.28.2.tgz",
-      "integrity": "sha512-sSATRjPeDBg3pdgHoQfoYBob11Kk1FGa9lui5RIHZCoCkJa9QKlvl3/vKz2usCmYYjs7ymJR/2Nnsqe+Hjt5nw==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/netbsd-arm64/-/netbsd-arm64-0.27.7.tgz",
+      "integrity": "sha512-b6pqtrQdigZBwZxAn1UpazEisvwaIDvdbMbmrly7cDTMFnw/+3lVxxCTGOrkPVnsYIosJJXAsILG9XcQS+Yu6w==",
       "cpu": [
         "arm64"
       ],
@@ -842,9 +836,9 @@
       }
     },
     "node_modules/@esbuild/netbsd-x64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/netbsd-x64/-/netbsd-x64-0.28.2.tgz",
-      "integrity": "sha512-lqnzCV+mM0gIADaKihiCg6ifgfU2L3h5E33rNQBN1Y4MaVGnzryzmvvf7UHxprpQdE8hpqLolJ9Rl+SkIRDpyw==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/netbsd-x64/-/netbsd-x64-0.27.7.tgz",
+      "integrity": "sha512-OfatkLojr6U+WN5EDYuoQhtM+1xco+/6FSzJJnuWiUw5eVcicbyK3dq5EeV/QHT1uy6GoDhGbFpprUiHUYggrw==",
       "cpu": [
         "x64"
       ],
@@ -859,9 +853,9 @@
       }
     },
     "node_modules/@esbuild/openbsd-arm64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/openbsd-arm64/-/openbsd-arm64-0.28.2.tgz",
-      "integrity": "sha512-AL2qJILH7lNjrDmCQDvdxMfAUIv8KMNZOvrwAQ8i8//ntL9FflhOyMJ8OZSMBb8/AWXe3/5v5S20y3zCoZWKoQ==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/openbsd-arm64/-/openbsd-arm64-0.27.7.tgz",
+      "integrity": "sha512-AFuojMQTxAz75Fo8idVcqoQWEHIXFRbOc1TrVcFSgCZtQfSdc1RXgB3tjOn/krRHENUB4j00bfGjyl2mJrU37A==",
       "cpu": [
         "arm64"
       ],
@@ -876,9 +870,9 @@
       }
     },
     "node_modules/@esbuild/openbsd-x64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/openbsd-x64/-/openbsd-x64-0.28.2.tgz",
-      "integrity": "sha512-QtiuPytchRyC4rwUKhexJdQKvDuZ6hWloi3igqPQNUJCS1/v9EiO3UTOXR6A3FoMo4fnAKbWJdqaIwhOzh8qEw==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/openbsd-x64/-/openbsd-x64-0.27.7.tgz",
+      "integrity": "sha512-+A1NJmfM8WNDv5CLVQYJ5PshuRm/4cI6WMZRg1by1GwPIQPCTs1GLEUHwiiQGT5zDdyLiRM/l1G0Pv54gvtKIg==",
       "cpu": [
         "x64"
       ],
@@ -893,9 +887,9 @@
       }
     },
     "node_modules/@esbuild/openharmony-arm64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/openharmony-arm64/-/openharmony-arm64-0.28.2.tgz",
-      "integrity": "sha512-WkhYDmpTjLvGlScA1rwjRUmhl4k8oXR3cIbtqWmELgU/dFeHHlEllxDvdWcNJV9rbzCexB5vz8gtNewWLgCT7Q==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/openharmony-arm64/-/openharmony-arm64-0.27.7.tgz",
+      "integrity": "sha512-+KrvYb/C8zA9CU/g0sR6w2RBw7IGc5J2BPnc3dYc5VJxHCSF1yNMxTV5LQ7GuKteQXZtspjFbiuW5/dOj7H4Yw==",
       "cpu": [
         "arm64"
       ],
@@ -910,9 +904,9 @@
       }
     },
     "node_modules/@esbuild/sunos-x64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/sunos-x64/-/sunos-x64-0.28.2.tgz",
-      "integrity": "sha512-GPMSkTOtMnv2U2F8gxe4Io6qmVs+YKyp832Etqqxr0hFngmXQ3rzwytelm3GIn7T4VviRUlf3sOgBOiTdvaf7g==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/sunos-x64/-/sunos-x64-0.27.7.tgz",
+      "integrity": "sha512-ikktIhFBzQNt/QDyOL580ti9+5mL/YZeUPKU2ivGtGjdTYoqz6jObj6nOMfhASpS4GU4Q/Clh1QtxWAvcYKamA==",
       "cpu": [
         "x64"
       ],
@@ -927,9 +921,9 @@
       }
     },
     "node_modules/@esbuild/win32-arm64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/win32-arm64/-/win32-arm64-0.28.2.tgz",
-      "integrity": "sha512-PIhhEkE9uPBleRBrQEJpUn7MBnibZzbGzYWPmY3x+YoVg/95zbjB4CxPPOQ8l5tYYM4mMaCthF8/1DIfBQQyWQ==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/win32-arm64/-/win32-arm64-0.27.7.tgz",
+      "integrity": "sha512-7yRhbHvPqSpRUV7Q20VuDwbjW5kIMwTHpptuUzV+AA46kiPze5Z7qgt6CLCK3pWFrHeNfDd1VKgyP4O+ng17CA==",
       "cpu": [
         "arm64"
       ],
@@ -944,9 +938,9 @@
       }
     },
     "node_modules/@esbuild/win32-ia32": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/win32-ia32/-/win32-ia32-0.28.2.tgz",
-      "integrity": "sha512-YmJbfTlvU7Sdn9BB+4PRES4oB6pxgS37MAONj+hBr/cpXS1aBPKXxNnDbu+QCWPj0o9dgyxeq79g6c5P8KeuYA==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/win32-ia32/-/win32-ia32-0.27.7.tgz",
+      "integrity": "sha512-SmwKXe6VHIyZYbBLJrhOoCJRB/Z1tckzmgTLfFYOfpMAx63BJEaL9ExI8x7v0oAO3Zh6D/Oi1gVxEYr5oUCFhw==",
       "cpu": [
         "ia32"
       ],
@@ -961,9 +955,9 @@
       }
     },
     "node_modules/@esbuild/win32-x64": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/@esbuild/win32-x64/-/win32-x64-0.28.2.tgz",
-      "integrity": "sha512-5ebpxr3nWMzrL/rnUI755Jkuee0bHL/Gq0WTF9lvcpv73wAp5eu8MfBUgWK9bhWvZjj7yX8etf/8tI8Ney695g==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/@esbuild/win32-x64/-/win32-x64-0.27.7.tgz",
+      "integrity": "sha512-56hiAJPhwQ1R4i+21FVF7V8kSD5zZTdHcVuRFMW0hn753vVfQN8xlx4uOPT4xoGH0Z/oVATuR82AiqSTDIpaHg==",
       "cpu": [
         "x64"
       ],
@@ -1121,24 +1115,6 @@
         "node": "^18.18.0 || ^20.9.0 || >=21.1.0"
       }
     },
-    "node_modules/@exodus/bytes": {
-      "version": "1.15.1",
-      "resolved": "https://registry.npmjs.org/@exodus/bytes/-/bytes-1.15.1.tgz",
-      "integrity": "sha512-S6mL0yNB/Abt9Ei4tq8gDhcczc4S3+vQ4ra7vxnAf+YHC02srtqxKKZghx2Dq6p0e66THKwR6r8N6P95wEty7Q==",
-      "dev": true,
-      "license": "MIT",
-      "engines": {
-        "node": "^20.19.0 || ^22.12.0 || >=24.0.0"
-      },
-      "peerDependencies": {
-        "@noble/hashes": "^1.8.0 || ^2.0.0"
-      },
-      "peerDependenciesMeta": {
-        "@noble/hashes": {
-          "optional": true
-        }
-      }
-    },
     "node_modules/@humanfs/core": {
       "version": "0.19.2",
       "resolved": "https://registry.npmjs.org/@humanfs/core/-/core-0.19.2.tgz",
@@ -1721,26 +1697,6 @@
         "@jridgewell/sourcemap-codec": "^1.4.14"
       }
     },
-    "node_modules/@napi-rs/lzma-linux-x64-gnu": {
-      "version": "1.5.1",
-      "resolved": "https://registry.npmjs.org/@napi-rs/lzma-linux-x64-gnu/-/lzma-linux-x64-gnu-1.5.1.tgz",
-      "integrity": "sha512-oTXEIha4SsuXdTA4Iyskj0kpdx2yVXdhd75c2v3xGrHFfVMsbhTPZU/nMPL4sWKo4pBHm3aucLaqGlF696dTyQ==",
-      "cpu": [
-        "x64"
-      ],
-      "dev": true,
-      "libc": [
-        "glibc"
-      ],
-      "license": "MIT",
-      "optional": true,
-      "os": [
-        "linux"
-      ],
-      "engines": {
-        "node": "^22.20 || ^24.12 || >=25"
-      }
-    },
     "node_modules/@napi-rs/wasm-runtime": {
       "version": "0.2.12",
       "resolved": "https://registry.npmjs.org/@napi-rs/wasm-runtime/-/wasm-runtime-0.2.12.tgz",
@@ -1946,10 +1902,26 @@
         "node": ">=12.4.0"
       }
     },
+    "node_modules/@playwright/test": {
+      "version": "1.59.1",
+      "resolved": "https://registry.npmjs.org/@playwright/test/-/test-1.59.1.tgz",
+      "integrity": "sha512-PG6q63nQg5c9rIi4/Z5lR5IVF7yU5MqmKaPOe0HSc0O2cX1fPi96sUQu5j7eo4gKCkB2AnNGoWt7y4/Xx3Kcqg==",
+      "devOptional": true,
+      "license": "Apache-2.0",
+      "dependencies": {
+        "playwright": "1.59.1"
+      },
+      "bin": {
+        "playwright": "cli.js"
+      },
+      "engines": {
+        "node": ">=18"
+      }
+    },
     "node_modules/@rollup/rollup-android-arm-eabi": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-android-arm-eabi/-/rollup-android-arm-eabi-4.62.4.tgz",
-      "integrity": "sha512-RrPokAb7dmbxFoeO3TloqHyOjgye8RkBhSqmp4aJMIex4c9r46ZstPnleDQOq1t46VOVjwIuwNogIqbodV1Vvg==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-android-arm-eabi/-/rollup-android-arm-eabi-4.60.3.tgz",
+      "integrity": "sha512-x35CNW/ANXG3hE/EZpRU8MXX1JDN86hBb2wMGAtltkz7pc6cxgjpy1OMMfDosOQ+2hWqIkag/fGok1Yady9nGw==",
       "cpu": [
         "arm"
       ],
@@ -1961,9 +1933,9 @@
       ]
     },
     "node_modules/@rollup/rollup-android-arm64": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-android-arm64/-/rollup-android-arm64-4.62.4.tgz",
-      "integrity": "sha512-JKuJc+pnpks2pjy7L/N3v/cAkZxYlnmuZoD840ldbMI5KDbC4iO9NKwPKYdjYFCMAIIlBzYSFHxIJVYzRo2/8A==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-android-arm64/-/rollup-android-arm64-4.60.3.tgz",
+      "integrity": "sha512-xw3xtkDApIOGayehp2+Rz4zimfkaX65r4t47iy+ymQB2G4iJCBBfj0ogVg5jpvjpn8UWn/+q9tprxleYeNp3Hw==",
       "cpu": [
         "arm64"
       ],
@@ -1975,9 +1947,9 @@
       ]
     },
     "node_modules/@rollup/rollup-darwin-arm64": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-darwin-arm64/-/rollup-darwin-arm64-4.62.4.tgz",
-      "integrity": "sha512-krw5uS2STmvJ02x0uTXHbqQNuz+9eZ1iw+qXk9dmW2gvV4jV7O2hEoOnuhFrpOPiel1mBFtqbxYZZtC46hXLOw==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-darwin-arm64/-/rollup-darwin-arm64-4.60.3.tgz",
+      "integrity": "sha512-vo6Y5Qfpx7/5EaamIwi0WqW2+zfiusVihKatLvtN1VFVy3D13uERk/6gZLU1UiHRL6fDXqj/ELIeVRGnvcTE1g==",
       "cpu": [
         "arm64"
       ],
@@ -1989,9 +1961,9 @@
       ]
     },
     "node_modules/@rollup/rollup-darwin-x64": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-darwin-x64/-/rollup-darwin-x64-4.62.4.tgz",
-      "integrity": "sha512-wsTxtgApb4PrOsNJIm0FZ1h3WvCC+k9uxLJ4ad75hgoS4NiRes2SoJFlDAyMwiUY8IssDqGcHbXuN0sx1tfF1A==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-darwin-x64/-/rollup-darwin-x64-4.60.3.tgz",
+      "integrity": "sha512-D+0QGcZhBzTN82weOnsSlY7V7+RMmPuF1CkbxyMAGE8+ZHeUjyb76ZiWmBlCu//AQQONvxcqRbwZTajZKqjuOw==",
       "cpu": [
         "x64"
       ],
@@ -2003,9 +1975,9 @@
       ]
     },
     "node_modules/@rollup/rollup-freebsd-arm64": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-freebsd-arm64/-/rollup-freebsd-arm64-4.62.4.tgz",
-      "integrity": "sha512-GUOnQlyZe3yAXhWOtOMsn5Qkrv5E5mZXa0thbARWi5Ei2szlVXJFQhddZ4HbAzh8q92w5twp+CQvs/eFanz9YQ==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-freebsd-arm64/-/rollup-freebsd-arm64-4.60.3.tgz",
+      "integrity": "sha512-6HnvHCT7fDyj6R0Ph7A6x8dQS/S38MClRWeDLqc0MdfWkxjiu1HSDYrdPhqSILzjTIC/pnXbbJbo+ft+gy/9hQ==",
       "cpu": [
         "arm64"
       ],
@@ -2017,9 +1989,9 @@
       ]
     },
     "node_modules/@rollup/rollup-freebsd-x64": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-freebsd-x64/-/rollup-freebsd-x64-4.62.4.tgz",
-      "integrity": "sha512-/Y7f3QuxjzPKsjA/rfEDa3+0vXqyjmJ50Ln8dPpCmWkKTrUoWHG1cWhTqaAMLob2m2nESWuC7yGrREz019Ztqg==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-freebsd-x64/-/rollup-freebsd-x64-4.60.3.tgz",
+      "integrity": "sha512-KHLgC3WKlUYW3ShFKnnosZDOJ0xjg9zp7au3sIm2bs/tGBeC2ipmvRh/N7JKi0t9Ue20C0dpEshi8WUubg+cnA==",
       "cpu": [
         "x64"
       ],
@@ -2031,16 +2003,13 @@
       ]
     },
     "node_modules/@rollup/rollup-linux-arm-gnueabihf": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm-gnueabihf/-/rollup-linux-arm-gnueabihf-4.62.4.tgz",
-      "integrity": "sha512-81wiiX3v7aqy+T+bT61TJ78yJjRquqFFTTbAPt08imfQQzkPIW8t6aJbkTagtCCrXMNc9D66+geqlK7ydLPNqA==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm-gnueabihf/-/rollup-linux-arm-gnueabihf-4.60.3.tgz",
+      "integrity": "sha512-DV6fJoxEYWJOvaZIsok7KrYl0tPvga5OZ2yvKHNNYyk/2roMLqQAbGhr78EQ5YhHpnhLKJD3S1WFusAkmUuV5g==",
       "cpu": [
         "arm"
       ],
       "dev": true,
-      "libc": [
-        "glibc"
-      ],
       "license": "MIT",
       "optional": true,
       "os": [
@@ -2048,16 +2017,13 @@
       ]
     },
     "node_modules/@rollup/rollup-linux-arm-musleabihf": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm-musleabihf/-/rollup-linux-arm-musleabihf-4.62.4.tgz",
-      "integrity": "sha512-9kmDIvNZqdoHOBZgNtpTBeLWYO/LVipM3H/j62P8848/l/VPEQL6N3uxU9pvP1oZAsXyC2MEnFP3ovRjo7WYNQ==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm-musleabihf/-/rollup-linux-arm-musleabihf-4.60.3.tgz",
+      "integrity": "sha512-mQKoJAzvuOs6F+TZybQO4GOTSMUu7v0WdxEk24krQ/uUxXoPTtHjuaUuPmFhtBcM4K0ons8nrE3JyhTuCFtT/w==",
       "cpu": [
         "arm"
       ],
       "dev": true,
-      "libc": [
-        "musl"
-      ],
       "license": "MIT",
       "optional": true,
       "os": [
@@ -2065,16 +2031,13 @@
       ]
     },
     "node_modules/@rollup/rollup-linux-arm64-gnu": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm64-gnu/-/rollup-linux-arm64-gnu-4.62.4.tgz",
-      "integrity": "sha512-CcnXHWnXg69g+DX5VWL3FHts3qMRN2uVEHX+BZvGLdd07/gXkn3ePjYtO1LDJvxkGKVHMclKBRa1QUTH+6toYQ==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm64-gnu/-/rollup-linux-arm64-gnu-4.60.3.tgz",
+      "integrity": "sha512-Whjj2qoiJ6+OOJMGptTYazaJvjOJm+iKHpXQM1P3LzGjt7Ff++Tp7nH4N8J/BUA7R9IHfDyx4DJIflifwnbmIA==",
       "cpu": [
         "arm64"
       ],
       "dev": true,
-      "libc": [
-        "glibc"
-      ],
       "license": "MIT",
       "optional": true,
       "os": [
@@ -2082,16 +2045,13 @@
       ]
     },
     "node_modules/@rollup/rollup-linux-arm64-musl": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm64-musl/-/rollup-linux-arm64-musl-4.62.4.tgz",
-      "integrity": "sha512-iFOibiHnTRuhrWLlRsOQFdZJJIa7S8OwkneJr4ocALP16u5yk6lWLINFwhHaEqBFMsKDUZofLkGos7+CPzGB3g==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm64-musl/-/rollup-linux-arm64-musl-4.60.3.tgz",
+      "integrity": "sha512-4YTNHKqGng5+yiZt3mg77nmyuCfmNfX4fPmyUapBcIk+BdwSwmCWGXOUxhXbBEkFHtoN5boLj/5NON+u5QC9tg==",
       "cpu": [
         "arm64"
       ],
       "dev": true,
-      "libc": [
-        "musl"
-      ],
       "license": "MIT",
       "optional": true,
       "os": [
@@ -2099,16 +2059,13 @@
       ]
     },
     "node_modules/@rollup/rollup-linux-loong64-gnu": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-loong64-gnu/-/rollup-linux-loong64-gnu-4.62.4.tgz",
-      "integrity": "sha512-XnWYMI7euHlb5a871xPja+Gm7DRCFU+FGRrtS2sMq9N8FvqtpagUy6gD4YOemC5MRk9xbh8+jYMEJbigFQwsgA==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-loong64-gnu/-/rollup-linux-loong64-gnu-4.60.3.tgz",
+      "integrity": "sha512-SU3kNlhkpI4UqlUc2VXPGK9o886ZsSeGfMAX2ba2b8DKmMXq4AL7KUrkSWVbb7koVqx41Yczx6dx5PNargIrEA==",
       "cpu": [
         "loong64"
       ],
       "dev": true,
-      "libc": [
-        "glibc"
-      ],
       "license": "MIT",
       "optional": true,
       "os": [
@@ -2116,16 +2073,13 @@
       ]
     },
     "node_modules/@rollup/rollup-linux-loong64-musl": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-loong64-musl/-/rollup-linux-loong64-musl-4.62.4.tgz",
-      "integrity": "sha512-qGDAlO0U8xedCcsdRm9oaoQY8DAx/QT7uIxJWhCdx0ceIWX783UC9QSYkdpzAe29wNiVfp24+bZdQmn49o45SQ==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-loong64-musl/-/rollup-linux-loong64-musl-4.60.3.tgz",
+      "integrity": "sha512-6lDLl5h4TXpB1mTf2rQWnAk/LcXrx9vBfu/DT5TIPhvMhRWaZ5MxkIc8u4lJAmBo6klTe1ywXIUHFjylW505sg==",
       "cpu": [
         "loong64"
       ],
       "dev": true,
-      "libc": [
-        "musl"
-      ],
       "license": "MIT",
       "optional": true,
       "os": [
@@ -2133,16 +2087,13 @@
       ]
     },
     "node_modules/@rollup/rollup-linux-ppc64-gnu": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-ppc64-gnu/-/rollup-linux-ppc64-gnu-4.62.4.tgz",
-      "integrity": "sha512-ru4H6ezD7ysA5EiEK6qkkaEb4modH8CTej6kUy/gQi20u3kB3G7Zn8snXXkeJSCOFKG/rbPPtM/+9Wgas1961w==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-ppc64-gnu/-/rollup-linux-ppc64-gnu-4.60.3.tgz",
+      "integrity": "sha512-BMo8bOw8evlup/8G+cj5xWtPyp93xPdyoSN16Zy90Q2QZ0ZYRhCt6ZJSwbrRzG9HApFabjwj2p25TUPDWrhzqQ==",
       "cpu": [
         "ppc64"
       ],
       "dev": true,
-      "libc": [
-        "glibc"
-      ],
       "license": "MIT",
       "optional": true,
       "os": [
@@ -2150,16 +2101,13 @@
       ]
     },
     "node_modules/@rollup/rollup-linux-ppc64-musl": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-ppc64-musl/-/rollup-linux-ppc64-musl-4.62.4.tgz",
-      "integrity": "sha512-2W4MO5WQVJnbJaZdvDb9rhBDuFU1nKIepPFpJUBsTh2k1YY2g+ODViaWuyOAjQ5cOP7NvrvLzt3wvHOoiAvc7w==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-ppc64-musl/-/rollup-linux-ppc64-musl-4.60.3.tgz",
+      "integrity": "sha512-E0L8X1dZN1/Rph+5VPF6Xj2G7JJvMACVXtamTJIDrVI44Y3K+G8gQaMEAavbqCGTa16InptiVrX6eM6pmJ+7qA==",
       "cpu": [
         "ppc64"
       ],
       "dev": true,
-      "libc": [
-        "musl"
-      ],
       "license": "MIT",
       "optional": true,
       "os": [
@@ -2167,16 +2115,13 @@
       ]
     },
     "node_modules/@rollup/rollup-linux-riscv64-gnu": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-riscv64-gnu/-/rollup-linux-riscv64-gnu-4.62.4.tgz",
-      "integrity": "sha512-+fxjfuoAmVMCYV5QyjoIpu0cp5DOiOTeqYFk1AVaxGr+/ravWLX89XfQmptsoWcaVy/TGf2hexzbUOrCQIL1CQ==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-riscv64-gnu/-/rollup-linux-riscv64-gnu-4.60.3.tgz",
+      "integrity": "sha512-oZJ/WHaVfHUiRAtmTAeo3DcevNsVvH8mbvodjZy7D5QKvCefO371SiKRpxoDcCxB3PTRTLayWBkvmDQKTcX/sw==",
       "cpu": [
         "riscv64"
       ],
       "dev": true,
-      "libc": [
-        "glibc"
-      ],
       "license": "MIT",
       "optional": true,
       "os": [
@@ -2184,16 +2129,13 @@
       ]
     },
     "node_modules/@rollup/rollup-linux-riscv64-musl": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-riscv64-musl/-/rollup-linux-riscv64-musl-4.62.4.tgz",
-      "integrity": "sha512-jTn8JfHGL4djjFxPuM06LmNUJDsst2jeVlsd9OmIH6zc5sC9K6rIuO4YajXatLUpBmBKl6b35ro1QZocLi+tcA==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-riscv64-musl/-/rollup-linux-riscv64-musl-4.60.3.tgz",
+      "integrity": "sha512-Dhbyh7j9FybM3YaTgaHmVALwA8AkUwTPccyCQ79TG9AJUsMQqgN1DDEZNr4+QUfwiWvLDumW5vdwzoeUF+TNxQ==",
       "cpu": [
         "riscv64"
       ],
       "dev": true,
-      "libc": [
-        "musl"
-      ],
       "license": "MIT",
       "optional": true,
       "os": [
@@ -2201,16 +2143,13 @@
       ]
     },
     "node_modules/@rollup/rollup-linux-s390x-gnu": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-s390x-gnu/-/rollup-linux-s390x-gnu-4.62.4.tgz",
-      "integrity": "sha512-oCJCJL4pXsoDcP2QZ+JVlPTIRc6266zsIaeJJsWImmF7HO0W8nb6HuSgZlMWxJwaPf8ehbSw8yo0EUw925hKsA==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-s390x-gnu/-/rollup-linux-s390x-gnu-4.60.3.tgz",
+      "integrity": "sha512-cJd1X5XhHHlltkaypz1UcWLA8AcoIi1aWhsvaWDskD1oz2eKCypnqvTQ8ykMNI0RSmm7NkTdSqSSD7zM0xa6Ig==",
       "cpu": [
         "s390x"
       ],
       "dev": true,
-      "libc": [
-        "glibc"
-      ],
       "license": "MIT",
       "optional": true,
       "os": [
@@ -2218,16 +2157,13 @@
       ]
     },
     "node_modules/@rollup/rollup-linux-x64-gnu": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-x64-gnu/-/rollup-linux-x64-gnu-4.62.4.tgz",
-      "integrity": "sha512-W69hukhZ3KKNRCaMIEzKvcFye42hh0FE1+YoYaf5+Ikacuftoco6yO/xouz0hc5d5W/s3yBro5jRiuEE/Q5vUw==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-x64-gnu/-/rollup-linux-x64-gnu-4.60.3.tgz",
+      "integrity": "sha512-DAZDBHQfG2oQuhY7mc6I3/qB4LU2fQCjRvxbDwd/Jdvb9fypP4IJ4qmtu6lNjes6B531AI8cg1aKC2di97bUxA==",
       "cpu": [
         "x64"
       ],
       "dev": true,
-      "libc": [
-        "glibc"
-      ],
       "license": "MIT",
       "optional": true,
       "os": [
@@ -2235,16 +2171,13 @@
       ]
     },
     "node_modules/@rollup/rollup-linux-x64-musl": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-x64-musl/-/rollup-linux-x64-musl-4.62.4.tgz",
-      "integrity": "sha512-qiXbGG2jkjXhzXpsFZSR2Xpb8DN/UaxYsbb/STbuR/6fpaDgRmmaq1B/LmtF2wQFOFOSsK2jdE0RZ3a0zHn4QA==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-x64-musl/-/rollup-linux-x64-musl-4.60.3.tgz",
+      "integrity": "sha512-cRxsE8c13mZOh3vP+wLDxpQBRrOHDIGOWyDL93Sy0Ga8y515fBcC2pjUfFwUe5T7tqvTvWbCpg1URM/AXdWIXA==",
       "cpu": [
         "x64"
       ],
       "dev": true,
-      "libc": [
-        "musl"
-      ],
       "license": "MIT",
       "optional": true,
       "os": [
@@ -2252,9 +2185,9 @@
       ]
     },
     "node_modules/@rollup/rollup-openbsd-x64": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-openbsd-x64/-/rollup-openbsd-x64-4.62.4.tgz",
-      "integrity": "sha512-nWeM//hxv8mIo6jD7Hu4o48DVmV9pbV6gsKaWU+4NFyqHoPKwrkRiZGLKUhOBk8qNmDmpwFtPKg80Bo/Tn4xiQ==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-openbsd-x64/-/rollup-openbsd-x64-4.60.3.tgz",
+      "integrity": "sha512-QaWcIgRxqEdQdhJqW4DJctsH6HCmo5vHxY0krHSX4jMtOqfzC+dqDGuHM87bu4H8JBeibWx7jFz+h6/4C8wA5Q==",
       "cpu": [
         "x64"
       ],
@@ -2266,9 +2199,9 @@
       ]
     },
     "node_modules/@rollup/rollup-openharmony-arm64": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-openharmony-arm64/-/rollup-openharmony-arm64-4.62.4.tgz",
-      "integrity": "sha512-s62SQ/vgsRSvMwDkOEfTqfgASF0f26ZNaQuTA6Aok5lrikf89yI2W0gFHvZb2Jpgc6N8JnOKZgCK2iciO3CsxQ==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-openharmony-arm64/-/rollup-openharmony-arm64-4.60.3.tgz",
+      "integrity": "sha512-AaXwSvUi3QIPtroAUw1t5yHGIyqKEXwH54WUocFolZhpGDruJcs8c+xPNDRn4XiQsS7MEwnYsHW2l0MBLDMkWg==",
       "cpu": [
         "arm64"
       ],
@@ -2280,9 +2213,9 @@
       ]
     },
     "node_modules/@rollup/rollup-win32-arm64-msvc": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-arm64-msvc/-/rollup-win32-arm64-msvc-4.62.4.tgz",
-      "integrity": "sha512-J6wGf8TVGbXJq+HH+ttTvrcfNKPbuZecV6KT1B8I18BC5IURUh5kl4Yl5OEP5eFIUoI5BWxCsyYMhFsDx8kekw==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-arm64-msvc/-/rollup-win32-arm64-msvc-4.60.3.tgz",
+      "integrity": "sha512-65LAKM/bAWDqKNEelHlcHvm2V+Vfb8C6INFxQXRHCvaVN1rJfwr4NvdP4FyzUaLqWfaCGaadf6UbTm8xJeYfEg==",
       "cpu": [
         "arm64"
       ],
@@ -2294,9 +2227,9 @@
       ]
     },
     "node_modules/@rollup/rollup-win32-ia32-msvc": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-ia32-msvc/-/rollup-win32-ia32-msvc-4.62.4.tgz",
-      "integrity": "sha512-zmfrQd/0wu6oJs8Vq8KwY/YtsKSsLtKe/HwAP4Wqy8LhWjeT55fHRAkOhYQ12wI3ayS4Tt12d5CDRD7N96SAYQ==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-ia32-msvc/-/rollup-win32-ia32-msvc-4.60.3.tgz",
+      "integrity": "sha512-EEM2gyhBF5MFnI6vMKdX1LAosE627RGBzIoGMdLloPZkXrUN0Ckqgr2Qi8+J3zip/8NVVro3/FjB+tjhZUgUHA==",
       "cpu": [
         "ia32"
       ],
@@ -2308,9 +2241,9 @@
       ]
     },
     "node_modules/@rollup/rollup-win32-x64-gnu": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-x64-gnu/-/rollup-win32-x64-gnu-4.62.4.tgz",
-      "integrity": "sha512-qPzHqdj9rfUD+w79dtE07zi/kFwKyCJqplp5K5ygeLTp7jLpAoc16OAH39HSmRC9UpozaecsleI8uAdEj6v2yw==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-x64-gnu/-/rollup-win32-x64-gnu-4.60.3.tgz",
+      "integrity": "sha512-E5Eb5H/DpxaoXH++Qkv28RcUJboMopmdDUALBczvHMf7hNIxaDZqwY5lK12UK1BHacSmvupoEWGu+n993Z0y1A==",
       "cpu": [
         "x64"
       ],
@@ -2322,9 +2255,9 @@
       ]
     },
     "node_modules/@rollup/rollup-win32-x64-msvc": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-x64-msvc/-/rollup-win32-x64-msvc-4.62.4.tgz",
-      "integrity": "sha512-zD6NdeWEByGE9QF9vCrlJ5YQB4oq9q91kPZS37Jwj5hOkvR1lTBSpsKhKDw4IJtbQ35LsTS1HD9DZYGKIshU1Q==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-x64-msvc/-/rollup-win32-x64-msvc-4.60.3.tgz",
+      "integrity": "sha512-hPt/bgL5cE+Qp+/TPHBqptcAgPzgj46mPcg/16zNUmbQk0j+mOEQV/+Lqu8QRtDV3Ek95Q6FeFITpuhl6OTsAA==",
       "cpu": [
         "x64"
       ],
@@ -2747,9 +2680,9 @@
       "license": "MIT"
     },
     "node_modules/@types/estree": {
-      "version": "1.0.9",
-      "resolved": "https://registry.npmjs.org/@types/estree/-/estree-1.0.9.tgz",
-      "integrity": "sha512-GhdPgy1el4/ImP05X05Uw4cw2/M93BCUmnEvWZNStlCzEKME4Fkk+YpoA5OiHNQmoS7Cafb8Xa3Pya8m1Qrzeg==",
+      "version": "1.0.8",
+      "resolved": "https://registry.npmjs.org/@types/estree/-/estree-1.0.8.tgz",
+      "integrity": "sha512-dWHzHa2WqEXI/O1E9OjrocMTKJl2mSrEolh1Iomrv6U+JuNwaHXsXx9bLu5gG7BUWFIN0skIQJQ/L1rIex4X6w==",
       "dev": true,
       "license": "MIT"
     },
@@ -3362,15 +3295,15 @@
       ]
     },
     "node_modules/@vitest/expect": {
-      "version": "3.2.7",
-      "resolved": "https://registry.npmjs.org/@vitest/expect/-/expect-3.2.7.tgz",
-      "integrity": "sha512-E8eBXaKibuvH2pSZErOjdVb5vF4PbKYcrnluBTYxEk1l/VhhwZg1kZQsdtjq+CsF5CFydf2Rdkz7jDHKSisi3w==",
+      "version": "3.2.4",
+      "resolved": "https://registry.npmjs.org/@vitest/expect/-/expect-3.2.4.tgz",
+      "integrity": "sha512-Io0yyORnB6sikFlt8QW5K7slY4OjqNX9jmJQ02QDda8lyM6B5oNgVWoSoKPac8/kgnCUzuHQKrSLtu/uOqqrig==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
         "@types/chai": "^5.2.2",
-        "@vitest/spy": "3.2.7",
-        "@vitest/utils": "3.2.7",
+        "@vitest/spy": "3.2.4",
+        "@vitest/utils": "3.2.4",
         "chai": "^5.2.0",
         "tinyrainbow": "^2.0.0"
       },
@@ -3379,13 +3312,13 @@
       }
     },
     "node_modules/@vitest/mocker": {
-      "version": "3.2.7",
-      "resolved": "https://registry.npmjs.org/@vitest/mocker/-/mocker-3.2.7.tgz",
-      "integrity": "sha512-Trr0hYO9CM3Wj6ksWHRhK9IZpIY6wTMO5u/MqXurMxT57sWBaOPEtP3Oq60ihZuh5JsiagKfz95OcxdEP6dBrA==",
+      "version": "3.2.4",
+      "resolved": "https://registry.npmjs.org/@vitest/mocker/-/mocker-3.2.4.tgz",
+      "integrity": "sha512-46ryTE9RZO/rfDd7pEqFl7etuyzekzEhUbTW3BvmeO/BcCMEgq59BKhek3dXDWgAj4oMK6OZi+vRr1wPW6qjEQ==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
-        "@vitest/spy": "3.2.7",
+        "@vitest/spy": "3.2.4",
         "estree-walker": "^3.0.3",
         "magic-string": "^0.30.17"
       },
@@ -3406,9 +3339,9 @@
       }
     },
     "node_modules/@vitest/pretty-format": {
-      "version": "3.2.7",
-      "resolved": "https://registry.npmjs.org/@vitest/pretty-format/-/pretty-format-3.2.7.tgz",
-      "integrity": "sha512-KUHlwqVu0sRlhCdyPdQ/wBoTfRahjUky1MubOmYw9fWfIZy1gNoHpuaaQBPAaMaVYdQYHJLurzj8ECCj5OwTqA==",
+      "version": "3.2.4",
+      "resolved": "https://registry.npmjs.org/@vitest/pretty-format/-/pretty-format-3.2.4.tgz",
+      "integrity": "sha512-IVNZik8IVRJRTr9fxlitMKeJeXFFFN0JaB9PHPGQ8NKQbGpfjlTx9zO4RefN8gp7eqjNy8nyK3NZmBzOPeIxtA==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
@@ -3419,13 +3352,13 @@
       }
     },
     "node_modules/@vitest/runner": {
-      "version": "3.2.7",
-      "resolved": "https://registry.npmjs.org/@vitest/runner/-/runner-3.2.7.tgz",
-      "integrity": "sha512-sB9y4ovltoQP+WaUPwmSxO9WIg9Ig694Di5PalVPsYHklAdE027mehpWF2SQSVq+k6sFgaivbTjTJwZLSHbedA==",
+      "version": "3.2.4",
+      "resolved": "https://registry.npmjs.org/@vitest/runner/-/runner-3.2.4.tgz",
+      "integrity": "sha512-oukfKT9Mk41LreEW09vt45f8wx7DordoWUZMYdY/cyAk7w5TWkTRCNZYF7sX7n2wB7jyGAl74OxgwhPgKaqDMQ==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
-        "@vitest/utils": "3.2.7",
+        "@vitest/utils": "3.2.4",
         "pathe": "^2.0.3",
         "strip-literal": "^3.0.0"
       },
@@ -3434,13 +3367,13 @@
       }
     },
     "node_modules/@vitest/snapshot": {
-      "version": "3.2.7",
-      "resolved": "https://registry.npmjs.org/@vitest/snapshot/-/snapshot-3.2.7.tgz",
-      "integrity": "sha512-7C+MwShwtBSI5Buwoyg3s/iY1eHL9PKAf+O1wVh/TdnjXUtkoL/9YQtre90i4MtNXM6edP1wJ2zOBpfCyhIS7g==",
+      "version": "3.2.4",
+      "resolved": "https://registry.npmjs.org/@vitest/snapshot/-/snapshot-3.2.4.tgz",
+      "integrity": "sha512-dEYtS7qQP2CjU27QBC5oUOxLE/v5eLkGqPE0ZKEIDGMs4vKWe7IjgLOeauHsR0D5YuuycGRO5oSRXnwnmA78fQ==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
-        "@vitest/pretty-format": "3.2.7",
+        "@vitest/pretty-format": "3.2.4",
         "magic-string": "^0.30.17",
         "pathe": "^2.0.3"
       },
@@ -3449,9 +3382,9 @@
       }
     },
     "node_modules/@vitest/spy": {
-      "version": "3.2.7",
-      "resolved": "https://registry.npmjs.org/@vitest/spy/-/spy-3.2.7.tgz",
-      "integrity": "sha512-Q2eQGI6d2L/hBtZ0qNuKcAGid68XK6cv1xsoaIma6PaJhHPoqcEJhYpXZ/5myCMqkNgtP6UKuBhbc0nHKnrkuQ==",
+      "version": "3.2.4",
+      "resolved": "https://registry.npmjs.org/@vitest/spy/-/spy-3.2.4.tgz",
+      "integrity": "sha512-vAfasCOe6AIK70iP5UD11Ac4siNUNJ9i/9PZ3NKx07sG6sUxeag1LWdNrMWeKKYBLlzuK+Gn65Yd5nyL6ds+nw==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
@@ -3462,13 +3395,13 @@
       }
     },
     "node_modules/@vitest/utils": {
-      "version": "3.2.7",
-      "resolved": "https://registry.npmjs.org/@vitest/utils/-/utils-3.2.7.tgz",
-      "integrity": "sha512-x6BDOd7dyo3PFLY3I9/HJ25X/6OurhGXk2/B9gOZNPF7XDVjeBK4k01lQE5uvDpbuheErh91qYuE1E2OEjK3Rw==",
+      "version": "3.2.4",
+      "resolved": "https://registry.npmjs.org/@vitest/utils/-/utils-3.2.4.tgz",
+      "integrity": "sha512-fB2V0JFrQSMsCo9HiSq3Ezpdv4iYaXRG1Sx8edX3MwxfyNn83mKiGzOcH+Fkxt4MHxr3y42fQi1oeAInqgX2QA==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
-        "@vitest/pretty-format": "3.2.7",
+        "@vitest/pretty-format": "3.2.4",
         "loupe": "^3.1.4",
         "tinyrainbow": "^2.0.0"
       },
@@ -4107,9 +4040,9 @@
       }
     },
     "node_modules/cssstyle/node_modules/lru-cache": {
-      "version": "11.5.2",
-      "resolved": "https://registry.npmjs.org/lru-cache/-/lru-cache-11.5.2.tgz",
-      "integrity": "sha512-4pfM1Ff0x50o0tQwb5ucw/RzNyD0/YJME6IVcStalZuMWxdt3sR3huStTtxz4PUmvZfRguvDejasvQ2kifR11g==",
+      "version": "11.3.6",
+      "resolved": "https://registry.npmjs.org/lru-cache/-/lru-cache-11.3.6.tgz",
+      "integrity": "sha512-Gf/KoL3C/MlI7Bt0PGI9I+TeTC/I6r/csU58N4BSNc4lppLBeKsOdFYkK+dX0ABDUMJNfCHTyPpzwwO21Awd3A==",
       "dev": true,
       "license": "BlueOak-1.0.0",
       "engines": {
@@ -4569,9 +4502,9 @@
       }
     },
     "node_modules/esbuild": {
-      "version": "0.28.2",
-      "resolved": "https://registry.npmjs.org/esbuild/-/esbuild-0.28.2.tgz",
-      "integrity": "sha512-HKVLS8dvII+xoKW9kmqxbRKrnWEXfJJr/FZhhJmiqIB0e053QNYFqOBouTMO/k5sID4MvCiUCvv8b9M4h32wIA==",
+      "version": "0.27.7",
+      "resolved": "https://registry.npmjs.org/esbuild/-/esbuild-0.27.7.tgz",
+      "integrity": "sha512-IxpibTjyVnmrIQo5aqNpCgoACA/dTKLTlhMHihVHhdkxKyPO1uBBthumT0rdHmcsk9uMonIWS0m4FljWzILh3w==",
       "dev": true,
       "hasInstallScript": true,
       "license": "MIT",
@@ -4582,32 +4515,32 @@
         "node": ">=18"
       },
       "optionalDependencies": {
-        "@esbuild/aix-ppc64": "0.28.2",
-        "@esbuild/android-arm": "0.28.2",
-        "@esbuild/android-arm64": "0.28.2",
-        "@esbuild/android-x64": "0.28.2",
-        "@esbuild/darwin-arm64": "0.28.2",
-        "@esbuild/darwin-x64": "0.28.2",
-        "@esbuild/freebsd-arm64": "0.28.2",
-        "@esbuild/freebsd-x64": "0.28.2",
-        "@esbuild/linux-arm": "0.28.2",
-        "@esbuild/linux-arm64": "0.28.2",
-        "@esbuild/linux-ia32": "0.28.2",
-        "@esbuild/linux-loong64": "0.28.2",
-        "@esbuild/linux-mips64el": "0.28.2",
-        "@esbuild/linux-ppc64": "0.28.2",
-        "@esbuild/linux-riscv64": "0.28.2",
-        "@esbuild/linux-s390x": "0.28.2",
-        "@esbuild/linux-x64": "0.28.2",
-        "@esbuild/netbsd-arm64": "0.28.2",
-        "@esbuild/netbsd-x64": "0.28.2",
-        "@esbuild/openbsd-arm64": "0.28.2",
-        "@esbuild/openbsd-x64": "0.28.2",
-        "@esbuild/openharmony-arm64": "0.28.2",
-        "@esbuild/sunos-x64": "0.28.2",
-        "@esbuild/win32-arm64": "0.28.2",
-        "@esbuild/win32-ia32": "0.28.2",
-        "@esbuild/win32-x64": "0.28.2"
+        "@esbuild/aix-ppc64": "0.27.7",
+        "@esbuild/android-arm": "0.27.7",
+        "@esbuild/android-arm64": "0.27.7",
+        "@esbuild/android-x64": "0.27.7",
+        "@esbuild/darwin-arm64": "0.27.7",
+        "@esbuild/darwin-x64": "0.27.7",
+        "@esbuild/freebsd-arm64": "0.27.7",
+        "@esbuild/freebsd-x64": "0.27.7",
+        "@esbuild/linux-arm": "0.27.7",
+        "@esbuild/linux-arm64": "0.27.7",
+        "@esbuild/linux-ia32": "0.27.7",
+        "@esbuild/linux-loong64": "0.27.7",
+        "@esbuild/linux-mips64el": "0.27.7",
+        "@esbuild/linux-ppc64": "0.27.7",
+        "@esbuild/linux-riscv64": "0.27.7",
+        "@esbuild/linux-s390x": "0.27.7",
+        "@esbuild/linux-x64": "0.27.7",
+        "@esbuild/netbsd-arm64": "0.27.7",
+        "@esbuild/netbsd-x64": "0.27.7",
+        "@esbuild/openbsd-arm64": "0.27.7",
+        "@esbuild/openbsd-x64": "0.27.7",
+        "@esbuild/openharmony-arm64": "0.27.7",
+        "@esbuild/sunos-x64": "0.27.7",
+        "@esbuild/win32-arm64": "0.27.7",
+        "@esbuild/win32-ia32": "0.27.7",
+        "@esbuild/win32-x64": "0.27.7"
       }
     },
     "node_modules/escalade": {
@@ -5050,9 +4983,9 @@
       }
     },
     "node_modules/expect-type": {
-      "version": "1.4.0",
-      "resolved": "https://registry.npmjs.org/expect-type/-/expect-type-1.4.0.tgz",
-      "integrity": "sha512-KfYbmpRm0VbLjEvVa9yGwCi9GI34xvi7A/HXYWQO65CSD2u3MczUJSuwXKFIxlGsgBQizV9q5J9NHj4VG0n+pA==",
+      "version": "1.3.0",
+      "resolved": "https://registry.npmjs.org/expect-type/-/expect-type-1.3.0.tgz",
+      "integrity": "sha512-knvyeauYhqjOYvQ66MznSMs83wmHrCycNEN6Ao+2AeYEfxUIkuiVxdEa1qlGEPK+We3n0THiDciYSsCcgW/DoA==",
       "dev": true,
       "license": "Apache-2.0",
       "engines": {
@@ -5201,10 +5134,9 @@
       }
     },
     "node_modules/fsevents": {
-      "version": "2.3.3",
-      "resolved": "https://registry.npmjs.org/fsevents/-/fsevents-2.3.3.tgz",
-      "integrity": "sha512-5xoDfX+fL7faATnagmWPpbFtwh/R77WmMMqqHGS65C3vvB0YHrgF+B1YmZ3441tMj5n63k0212XNoJwzlhffQw==",
-      "dev": true,
+      "version": "2.3.2",
+      "resolved": "https://registry.npmjs.org/fsevents/-/fsevents-2.3.2.tgz",
+      "integrity": "sha512-xiqMQR4xAeHTuB9uWm+fFRcIOgKBMiOBP+eXiyT7jsgVCq1bkVygt00oASowB7EdtpOHaaPgKt812P9ab+DDKA==",
       "hasInstallScript": true,
       "license": "MIT",
       "optional": true,
@@ -5521,16 +5453,16 @@
       }
     },
     "node_modules/html-encoding-sniffer": {
-      "version": "6.0.0",
-      "resolved": "https://registry.npmjs.org/html-encoding-sniffer/-/html-encoding-sniffer-6.0.0.tgz",
-      "integrity": "sha512-CV9TW3Y3f8/wT0BRFc1/KAVQ3TUHiXmaAb6VW9vtiMFf7SLoMd1PdAc4W3KFOFETBJUb90KatHqlsZMWV+R9Gg==",
+      "version": "4.0.0",
+      "resolved": "https://registry.npmjs.org/html-encoding-sniffer/-/html-encoding-sniffer-4.0.0.tgz",
+      "integrity": "sha512-Y22oTqIU4uuPgEemfz7NDJz6OeKf12Lsu+QC+s3BVpda64lTiMYCyGwg5ki4vFxkMwQdeZDl2adZoqUgdFuTgQ==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
-        "@exodus/bytes": "^1.6.0"
+        "whatwg-encoding": "^3.1.1"
       },
       "engines": {
-        "node": "^20.19.0 || ^22.12.0 || >=24.0.0"
+        "node": ">=18"
       }
     },
     "node_modules/http-proxy-agent": {
@@ -5561,6 +5493,19 @@
         "node": ">= 14"
       }
     },
+    "node_modules/iconv-lite": {
+      "version": "0.6.3",
+      "resolved": "https://registry.npmjs.org/iconv-lite/-/iconv-lite-0.6.3.tgz",
+      "integrity": "sha512-4fCk79wshMdzMp2rH06qWrJE4iolqLhCUH+OiuIgU++RB0+94NlDL81atO7GX55uUKueo0txHNtvEyI6D7WdMw==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "safer-buffer": ">= 2.1.2 < 3.0.0"
+      },
+      "engines": {
+        "node": ">=0.10.0"
+      }
+    },
     "node_modules/ignore": {
       "version": "5.3.2",
       "resolved": "https://registry.npmjs.org/ignore/-/ignore-5.3.2.tgz",
@@ -6108,35 +6053,35 @@
       }
     },
     "node_modules/jsdom": {
-      "version": "27.4.0",
-      "resolved": "https://registry.npmjs.org/jsdom/-/jsdom-27.4.0.tgz",
-      "integrity": "sha512-mjzqwWRD9Y1J1KUi7W97Gja1bwOOM5Ug0EZ6UDK3xS7j7mndrkwozHtSblfomlzyB4NepioNt+B2sOSzczVgtQ==",
+      "version": "27.0.1",
+      "resolved": "https://registry.npmjs.org/jsdom/-/jsdom-27.0.1.tgz",
+      "integrity": "sha512-SNSQteBL1IlV2zqhwwolaG9CwhIhTvVHWg3kTss/cLE7H/X4644mtPQqYvCfsSrGQWt9hSZcgOXX8bOZaMN+kA==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
-        "@acemir/cssom": "^0.9.28",
-        "@asamuzakjp/dom-selector": "^6.7.6",
-        "@exodus/bytes": "^1.6.0",
-        "cssstyle": "^5.3.4",
+        "@asamuzakjp/dom-selector": "^6.7.2",
+        "cssstyle": "^5.3.1",
         "data-urls": "^6.0.0",
         "decimal.js": "^10.6.0",
-        "html-encoding-sniffer": "^6.0.0",
+        "html-encoding-sniffer": "^4.0.0",
         "http-proxy-agent": "^7.0.2",
         "https-proxy-agent": "^7.0.6",
         "is-potential-custom-element-name": "^1.0.1",
         "parse5": "^8.0.0",
+        "rrweb-cssom": "^0.8.0",
         "saxes": "^6.0.0",
         "symbol-tree": "^3.2.4",
         "tough-cookie": "^6.0.0",
         "w3c-xmlserializer": "^5.0.0",
         "webidl-conversions": "^8.0.0",
+        "whatwg-encoding": "^3.1.1",
         "whatwg-mimetype": "^4.0.0",
         "whatwg-url": "^15.1.0",
         "ws": "^8.18.3",
         "xml-name-validator": "^5.0.0"
       },
       "engines": {
-        "node": "^20.19.0 || ^22.12.0 || >=24.0.0"
+        "node": ">=20"
       },
       "peerDependencies": {
         "canvas": "^3.0.0"
@@ -7098,6 +7043,38 @@
         "url": "https://github.com/sponsors/jonschlinkert"
       }
     },
+    "node_modules/playwright": {
+      "version": "1.59.1",
+      "resolved": "https://registry.npmjs.org/playwright/-/playwright-1.59.1.tgz",
+      "integrity": "sha512-C8oWjPR3F81yljW9o5OxcWzfh6avkVwDD2VYdwIGqTkl+OGFISgypqzfu7dOe4QNLL2aqcWBmI3PMtLIK233lw==",
+      "devOptional": true,
+      "license": "Apache-2.0",
+      "dependencies": {
+        "playwright-core": "1.59.1"
+      },
+      "bin": {
+        "playwright": "cli.js"
+      },
+      "engines": {
+        "node": ">=18"
+      },
+      "optionalDependencies": {
+        "fsevents": "2.3.2"
+      }
+    },
+    "node_modules/playwright-core": {
+      "version": "1.59.1",
+      "resolved": "https://registry.npmjs.org/playwright-core/-/playwright-core-1.59.1.tgz",
+      "integrity": "sha512-HBV/RJg81z5BiiZ9yPzIiClYV/QMsDCKUyogwH9p3MCP6IYjUFu/MActgYAvK0oWyV9NlwM3GLBjADyWgydVyg==",
+      "devOptional": true,
+      "license": "Apache-2.0",
+      "bin": {
+        "playwright-core": "cli.js"
+      },
+      "engines": {
+        "node": ">=18"
+      }
+    },
     "node_modules/possible-typed-array-names": {
       "version": "1.1.0",
       "resolved": "https://registry.npmjs.org/possible-typed-array-names/-/possible-typed-array-names-1.1.0.tgz",
@@ -7380,13 +7357,13 @@
       }
     },
     "node_modules/rollup": {
-      "version": "4.62.4",
-      "resolved": "https://registry.npmjs.org/rollup/-/rollup-4.62.4.tgz",
-      "integrity": "sha512-RXOqwaPsBGjMNMa4sQjDjHieHEZDFoj/Rdr46l2MU5DfEs16wHJPC2RPTPHWhNl+M3aI472LLqFkFKut4SblOg==",
+      "version": "4.60.3",
+      "resolved": "https://registry.npmjs.org/rollup/-/rollup-4.60.3.tgz",
+      "integrity": "sha512-pAQK9HalE84QSm4Po3EmWIZPd3FnjkShVkiMlz1iligWYkWQ7wHYd1PF/T7QZ5TVSD6uSTon5gBVMSM4JfBV+A==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
-        "@types/estree": "1.0.9"
+        "@types/estree": "1.0.8"
       },
       "bin": {
         "rollup": "dist/bin/rollup"
@@ -7396,35 +7373,41 @@
         "npm": ">=8.0.0"
       },
       "optionalDependencies": {
-        "@napi-rs/lzma-linux-x64-gnu": "1.5.1",
-        "@rollup/rollup-android-arm-eabi": "4.62.4",
-        "@rollup/rollup-android-arm64": "4.62.4",
-        "@rollup/rollup-darwin-arm64": "4.62.4",
-        "@rollup/rollup-darwin-x64": "4.62.4",
-        "@rollup/rollup-freebsd-arm64": "4.62.4",
-        "@rollup/rollup-freebsd-x64": "4.62.4",
-        "@rollup/rollup-linux-arm-gnueabihf": "4.62.4",
-        "@rollup/rollup-linux-arm-musleabihf": "4.62.4",
-        "@rollup/rollup-linux-arm64-gnu": "4.62.4",
-        "@rollup/rollup-linux-arm64-musl": "4.62.4",
-        "@rollup/rollup-linux-loong64-gnu": "4.62.4",
-        "@rollup/rollup-linux-loong64-musl": "4.62.4",
-        "@rollup/rollup-linux-ppc64-gnu": "4.62.4",
-        "@rollup/rollup-linux-ppc64-musl": "4.62.4",
-        "@rollup/rollup-linux-riscv64-gnu": "4.62.4",
-        "@rollup/rollup-linux-riscv64-musl": "4.62.4",
-        "@rollup/rollup-linux-s390x-gnu": "4.62.4",
-        "@rollup/rollup-linux-x64-gnu": "4.62.4",
-        "@rollup/rollup-linux-x64-musl": "4.62.4",
-        "@rollup/rollup-openbsd-x64": "4.62.4",
-        "@rollup/rollup-openharmony-arm64": "4.62.4",
-        "@rollup/rollup-win32-arm64-msvc": "4.62.4",
-        "@rollup/rollup-win32-ia32-msvc": "4.62.4",
-        "@rollup/rollup-win32-x64-gnu": "4.62.4",
-        "@rollup/rollup-win32-x64-msvc": "4.62.4",
+        "@rollup/rollup-android-arm-eabi": "4.60.3",
+        "@rollup/rollup-android-arm64": "4.60.3",
+        "@rollup/rollup-darwin-arm64": "4.60.3",
+        "@rollup/rollup-darwin-x64": "4.60.3",
+        "@rollup/rollup-freebsd-arm64": "4.60.3",
+        "@rollup/rollup-freebsd-x64": "4.60.3",
+        "@rollup/rollup-linux-arm-gnueabihf": "4.60.3",
+        "@rollup/rollup-linux-arm-musleabihf": "4.60.3",
+        "@rollup/rollup-linux-arm64-gnu": "4.60.3",
+        "@rollup/rollup-linux-arm64-musl": "4.60.3",
+        "@rollup/rollup-linux-loong64-gnu": "4.60.3",
+        "@rollup/rollup-linux-loong64-musl": "4.60.3",
+        "@rollup/rollup-linux-ppc64-gnu": "4.60.3",
+        "@rollup/rollup-linux-ppc64-musl": "4.60.3",
+        "@rollup/rollup-linux-riscv64-gnu": "4.60.3",
+        "@rollup/rollup-linux-riscv64-musl": "4.60.3",
+        "@rollup/rollup-linux-s390x-gnu": "4.60.3",
+        "@rollup/rollup-linux-x64-gnu": "4.60.3",
+        "@rollup/rollup-linux-x64-musl": "4.60.3",
+        "@rollup/rollup-openbsd-x64": "4.60.3",
+        "@rollup/rollup-openharmony-arm64": "4.60.3",
+        "@rollup/rollup-win32-arm64-msvc": "4.60.3",
+        "@rollup/rollup-win32-ia32-msvc": "4.60.3",
+        "@rollup/rollup-win32-x64-gnu": "4.60.3",
+        "@rollup/rollup-win32-x64-msvc": "4.60.3",
         "fsevents": "~2.3.2"
       }
     },
+    "node_modules/rrweb-cssom": {
+      "version": "0.8.0",
+      "resolved": "https://registry.npmjs.org/rrweb-cssom/-/rrweb-cssom-0.8.0.tgz",
+      "integrity": "sha512-guoltQEx+9aMf2gDZ0s62EcV8lsXR+0w8915TC3ITdn2YueuNjdAYh/levpU9nFaoChh9RUS5ZdQMrKfVEN9tw==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/run-parallel": {
       "version": "1.2.0",
       "resolved": "https://registry.npmjs.org/run-parallel/-/run-parallel-1.2.0.tgz",
@@ -7504,6 +7487,13 @@
         "url": "https://github.com/sponsors/ljharb"
       }
     },
+    "node_modules/safer-buffer": {
+      "version": "2.1.2",
+      "resolved": "https://registry.npmjs.org/safer-buffer/-/safer-buffer-2.1.2.tgz",
+      "integrity": "sha512-YZo3K82SD7Riyi0E1EQPojLz7kpepnSQI9IyPbHHg1XXXevb5dJI7tpyN2ADxGcQbHG7vcyRHk0cbwqcQriUtg==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/saxes": {
       "version": "6.0.0",
       "resolved": "https://registry.npmjs.org/saxes/-/saxes-6.0.0.tgz",
@@ -8148,22 +8138,22 @@
       }
     },
     "node_modules/tldts": {
-      "version": "7.4.10",
-      "resolved": "https://registry.npmjs.org/tldts/-/tldts-7.4.10.tgz",
-      "integrity": "sha512-GgouD1B+sWwvkaEq8vXC15DjQitxbvs12oIXELpconwm+Tg3zfcEv4jgzq3vtKverDXsg3VI8aRgNL2Nra0Iog==",
+      "version": "7.0.30",
+      "resolved": "https://registry.npmjs.org/tldts/-/tldts-7.0.30.tgz",
+      "integrity": "sha512-ELrFxuqsDdHUwoh0XxDbxuLD3Wnz49Z57IFvTtvWy1hJdcMZjXLIuonjilCiWHlT2GbE4Wlv1wKVTzDFnXH1aw==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
-        "tldts-core": "^7.4.10"
+        "tldts-core": "^7.0.30"
       },
       "bin": {
         "tldts": "bin/cli.js"
       }
     },
     "node_modules/tldts-core": {
-      "version": "7.4.10",
-      "resolved": "https://registry.npmjs.org/tldts-core/-/tldts-core-7.4.10.tgz",
-      "integrity": "sha512-KnQjp53ZekKgm/r3l+u8kJGGzYgrWdP8+Mql7a4vijh2WE0IrZWspQj/TpTxDho/YxO+AnOZnIjQcCD+q6iJsw==",
+      "version": "7.0.30",
+      "resolved": "https://registry.npmjs.org/tldts-core/-/tldts-core-7.0.30.tgz",
+      "integrity": "sha512-uiHN8PIB1VmWyS98eZYja4xzlYqeFZVjb4OuYlJQnZAuJhMw4PbKQOKgHKhBdJR3FE/t5mUQ1Kd80++B+qhD1Q==",
       "dev": true,
       "license": "MIT"
     },
@@ -8181,9 +8171,9 @@
       }
     },
     "node_modules/tough-cookie": {
-      "version": "6.0.2",
-      "resolved": "https://registry.npmjs.org/tough-cookie/-/tough-cookie-6.0.2.tgz",
-      "integrity": "sha512-exgYmnmL/sJpR3upZfXG5PoatXQii55xAiXGXzY+sROLZ/Y+SLcp9PgJNI9Vz37HpQ74WvDcLT8eqm+kV3FzrA==",
+      "version": "6.0.1",
+      "resolved": "https://registry.npmjs.org/tough-cookie/-/tough-cookie-6.0.1.tgz",
+      "integrity": "sha512-LktZQb3IeoUWB9lqR5EWTHgW/VTITCXg4D21M+lvybRVdylLrRMnqaIONLVb5mav8vM19m44HIcGq4qASeu2Qw==",
       "dev": true,
       "license": "BSD-3-Clause",
       "dependencies": {
@@ -8252,9 +8242,9 @@
       "license": "0BSD"
     },
     "node_modules/tsx": {
-      "version": "4.23.12",
-      "resolved": "https://registry.npmjs.org/tsx/-/tsx-4.23.12.tgz",
-      "integrity": "sha512-FDf4L4sYzKtzWYhU/Xm0AQFdTjdIxNo9ElTf2mxXM6k8YMHXzYUe4yODVaXP4V9uMFbVg8c0qyBccK2OOxb45Q==",
+      "version": "4.23.0",
+      "resolved": "https://registry.npmjs.org/tsx/-/tsx-4.23.0.tgz",
+      "integrity": "sha512-eUdUIaCr963q2h5u3+QwvYp0+eqPvn+egeqZUm0hwERCqqx1E3kK5ehbGCvqSE5MQAULr67ww0cA3jKc3YkM1w==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
@@ -8270,121 +8260,620 @@
         "fsevents": "~2.3.3"
       }
     },
-    "node_modules/type-check": {
-      "version": "0.4.0",
-      "resolved": "https://registry.npmjs.org/type-check/-/type-check-0.4.0.tgz",
-      "integrity": "sha512-XleUoc9uwGXqjWwXaUTZAmzMcFZ5858QA2vvx1Ur5xIcixXIP+8LnFDgRplU30us6teqdlskFfu+ae4K79Ooew==",
+    "node_modules/tsx/node_modules/@esbuild/aix-ppc64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/aix-ppc64/-/aix-ppc64-0.28.1.tgz",
+      "integrity": "sha512-Svl7tq8k/08+p6CXPpRjQ1fKX+1odH/BQbb48fV6fj3CWHhsoIOoY87w1oHXm0qEpkIK3ZfVgp0hed3XBXzXMQ==",
+      "cpu": [
+        "ppc64"
+      ],
       "dev": true,
       "license": "MIT",
-      "dependencies": {
-        "prelude-ls": "^1.2.1"
-      },
+      "optional": true,
+      "os": [
+        "aix"
+      ],
       "engines": {
-        "node": ">= 0.8.0"
+        "node": ">=18"
       }
     },
-    "node_modules/typed-array-buffer": {
-      "version": "1.0.3",
-      "resolved": "https://registry.npmjs.org/typed-array-buffer/-/typed-array-buffer-1.0.3.tgz",
-      "integrity": "sha512-nAYYwfY3qnzX30IkA6AQZjVbtK6duGontcQm1WSG1MD94YLqK0515GNApXkoxKOWMusVssAHWLh9SeaoefYFGw==",
+    "node_modules/tsx/node_modules/@esbuild/android-arm": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/android-arm/-/android-arm-0.28.1.tgz",
+      "integrity": "sha512-0k2F129Xdio1TdJfzJ8sy1Q47vUD2NnwdhiAf7drUN1EBTfPf4hsFCtmMgu/6m8JSzsBrlmVjudMBQqOfG8usQ==",
+      "cpu": [
+        "arm"
+      ],
       "dev": true,
       "license": "MIT",
-      "dependencies": {
-        "call-bound": "^1.0.3",
-        "es-errors": "^1.3.0",
-        "is-typed-array": "^1.1.14"
-      },
+      "optional": true,
+      "os": [
+        "android"
+      ],
       "engines": {
-        "node": ">= 0.4"
+        "node": ">=18"
       }
     },
-    "node_modules/typed-array-byte-length": {
-      "version": "1.0.3",
-      "resolved": "https://registry.npmjs.org/typed-array-byte-length/-/typed-array-byte-length-1.0.3.tgz",
-      "integrity": "sha512-BaXgOuIxz8n8pIq3e7Atg/7s+DpiYrxn4vdot3w9KbnBhcRQq6o3xemQdIfynqSeXeDrF32x+WvfzmOjPiY9lg==",
+    "node_modules/tsx/node_modules/@esbuild/android-arm64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/android-arm64/-/android-arm64-0.28.1.tgz",
+      "integrity": "sha512-34EGEbCIAgosYz6goLcopX6Mo7NyGv9tfwEM2/7Ce2VcVRk568iSvniGWcUXIy7wEDR1wzolcxcriFVrWYcwBg==",
+      "cpu": [
+        "arm64"
+      ],
       "dev": true,
       "license": "MIT",
-      "dependencies": {
-        "call-bind": "^1.0.8",
-        "for-each": "^0.3.3",
-        "gopd": "^1.2.0",
-        "has-proto": "^1.2.0",
-        "is-typed-array": "^1.1.14"
-      },
+      "optional": true,
+      "os": [
+        "android"
+      ],
       "engines": {
-        "node": ">= 0.4"
-      },
-      "funding": {
-        "url": "https://github.com/sponsors/ljharb"
+        "node": ">=18"
       }
     },
-    "node_modules/typed-array-byte-offset": {
-      "version": "1.0.4",
-      "resolved": "https://registry.npmjs.org/typed-array-byte-offset/-/typed-array-byte-offset-1.0.4.tgz",
-      "integrity": "sha512-bTlAFB/FBYMcuX81gbL4OcpH5PmlFHqlCCpAl8AlEzMz5k53oNDvN8p1PNOWLEmI2x4orp3raOFB51tv9X+MFQ==",
+    "node_modules/tsx/node_modules/@esbuild/android-x64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/android-x64/-/android-x64-0.28.1.tgz",
+      "integrity": "sha512-dbwY7ltSMDWsRatcRpCnES4F+im88OCUgGZjy52shC7GqHRE/cYlxNbB4Z4UpJswpcc4Qxd2oE/ufM0p61IKng==",
+      "cpu": [
+        "x64"
+      ],
       "dev": true,
       "license": "MIT",
-      "dependencies": {
-        "available-typed-arrays": "^1.0.7",
-        "call-bind": "^1.0.8",
-        "for-each": "^0.3.3",
-        "gopd": "^1.2.0",
-        "has-proto": "^1.2.0",
-        "is-typed-array": "^1.1.15",
-        "reflect.getprototypeof": "^1.0.9"
-      },
+      "optional": true,
+      "os": [
+        "android"
+      ],
       "engines": {
-        "node": ">= 0.4"
-      },
-      "funding": {
-        "url": "https://github.com/sponsors/ljharb"
+        "node": ">=18"
       }
     },
-    "node_modules/typed-array-length": {
-      "version": "1.0.7",
-      "resolved": "https://registry.npmjs.org/typed-array-length/-/typed-array-length-1.0.7.tgz",
-      "integrity": "sha512-3KS2b+kL7fsuk/eJZ7EQdnEmQoaho/r6KUef7hxvltNA5DR8NAUM+8wJMbJyZ4G9/7i3v5zPBIMN5aybAh2/Jg==",
+    "node_modules/tsx/node_modules/@esbuild/darwin-arm64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/darwin-arm64/-/darwin-arm64-0.28.1.tgz",
+      "integrity": "sha512-TZbWkQY7kvTAXbXUT7uVACR5cMHsDiSz9z7ZKAX/RTq/WJEk3QyRr0wZpNhBDX+/0CtdqUIJlOiodQcta6tY3Q==",
+      "cpu": [
+        "arm64"
+      ],
       "dev": true,
       "license": "MIT",
-      "dependencies": {
-        "call-bind": "^1.0.7",
-        "for-each": "^0.3.3",
-        "gopd": "^1.0.1",
-        "is-typed-array": "^1.1.13",
-        "possible-typed-array-names": "^1.0.0",
-        "reflect.getprototypeof": "^1.0.6"
-      },
+      "optional": true,
+      "os": [
+        "darwin"
+      ],
       "engines": {
-        "node": ">= 0.4"
-      },
-      "funding": {
-        "url": "https://github.com/sponsors/ljharb"
+        "node": ">=18"
       }
     },
-    "node_modules/typescript": {
-      "version": "5.9.3",
-      "resolved": "https://registry.npmjs.org/typescript/-/typescript-5.9.3.tgz",
-      "integrity": "sha512-jl1vZzPDinLr9eUt3J/t7V6FgNEw9QjvBPdysz9KfQDD41fQrC2Y4vKQdiaUpFT4bXlb1RHhLpp8wtm6M5TgSw==",
+    "node_modules/tsx/node_modules/@esbuild/darwin-x64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/darwin-x64/-/darwin-x64-0.28.1.tgz",
+      "integrity": "sha512-zfdzgK9ACBNZLI/CyHTOx81SyNbM6YXn7rxSgX97VjyiPl9W1i4Ka4fgKECEoFCKGpvBj5qArWIGgQjOwkgskQ==",
+      "cpu": [
+        "x64"
+      ],
       "dev": true,
-      "license": "Apache-2.0",
-      "bin": {
-        "tsc": "bin/tsc",
-        "tsserver": "bin/tsserver"
-      },
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "darwin"
+      ],
       "engines": {
-        "node": ">=14.17"
+        "node": ">=18"
       }
     },
-    "node_modules/typescript-eslint": {
-      "version": "8.59.2",
-      "resolved": "https://registry.npmjs.org/typescript-eslint/-/typescript-eslint-8.59.2.tgz",
-      "integrity": "sha512-pJw051uomb3ZeCzGTpRb8RbEqB5Y4WWet8gl/GcTlU35BSx0PVdZ86/bqkQCyKKuraVQEK7r6kBHQXF+fBhkoQ==",
+    "node_modules/tsx/node_modules/@esbuild/freebsd-arm64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/freebsd-arm64/-/freebsd-arm64-0.28.1.tgz",
+      "integrity": "sha512-wG2EA8ENdEI0qhkSZMjfqrdY+ziCYCPMmtZjjIwOmXFjmyzEHn+UUxk5of+SYsjtfs3VpnlC7QLzSI5hY/rOAw==",
+      "cpu": [
+        "arm64"
+      ],
       "dev": true,
       "license": "MIT",
-      "dependencies": {
-        "@typescript-eslint/eslint-plugin": "8.59.2",
-        "@typescript-eslint/parser": "8.59.2",
-        "@typescript-eslint/typescript-estree": "8.59.2",
+      "optional": true,
+      "os": [
+        "freebsd"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/freebsd-x64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/freebsd-x64/-/freebsd-x64-0.28.1.tgz",
+      "integrity": "sha512-i7dZ9vQgnvSCzi/rYCXNgtF/U+eKZNJBzu3eTQbRgHnM7tNSizLOkRFAl3qzVc/Op/u5YkHHa4pf/3DOYHthLQ==",
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
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/linux-arm": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-arm/-/linux-arm-0.28.1.tgz",
+      "integrity": "sha512-qVXBOHQS+d5Y722GwJzJUtOLlX7km3CraOaGormF1pDtPd2C/l1SHRPgjLunLGe51Sh5YYWKMFDyV4SxgMQYTQ==",
+      "cpu": [
+        "arm"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/linux-arm64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-arm64/-/linux-arm64-0.28.1.tgz",
+      "integrity": "sha512-yHs+0uc8+nvEAfAfxrWQKK5peSNzBc4PegcMO0EJ2hT71uA7vB8Ihg2e77R2P7SG5uYjPbHlLLmve4LLLRCf0g==",
+      "cpu": [
+        "arm64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/linux-ia32": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-ia32/-/linux-ia32-0.28.1.tgz",
+      "integrity": "sha512-d1z4ZuP0ajrfz/FhGT4vv278rX8KnPPJx8i5+AtK7TYbx9Le9F1hyzurZpkEyjkGa9dUGhQow4C1NmeGvqxN2w==",
+      "cpu": [
+        "ia32"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/linux-loong64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-loong64/-/linux-loong64-0.28.1.tgz",
+      "integrity": "sha512-M5sRjUVZrkm1OAPR3dlOYzNmN+loZKGVi1VUQGrwuqLcbR6qeAz+famMhjASeH3YVKvZz+zT1jlh/keC3Rj/lg==",
+      "cpu": [
+        "loong64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/linux-mips64el": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-mips64el/-/linux-mips64el-0.28.1.tgz",
+      "integrity": "sha512-mRObBZeHh2OxcBFPWE/FjylkRgZdYuiTR3vaTozquCGOH14iP9oN4x4Ge81CoIDYQrXmIxpFumJBu5MtZpnQJQ==",
+      "cpu": [
+        "mips64el"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/linux-ppc64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-ppc64/-/linux-ppc64-0.28.1.tgz",
+      "integrity": "sha512-slScBsMAb3GFDcdrCgLwZtPYRoH2H/youv10QiZyRjmsP48fznoveWytSgCI/R0ZcUgpc0ZhIUEx6LHts8yrfQ==",
+      "cpu": [
+        "ppc64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/linux-riscv64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-riscv64/-/linux-riscv64-0.28.1.tgz",
+      "integrity": "sha512-kw0owk1o0GFETUJyW0jc0G4Yzs0BHZn0JDZ8JRT088vjJYX777BAs1fDGxAC+q831qOs2DTC96mNsG2opdfyyQ==",
+      "cpu": [
+        "riscv64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/linux-s390x": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-s390x/-/linux-s390x-0.28.1.tgz",
+      "integrity": "sha512-/lAIjX8aYFRByhh6L5rYtPEDRqa9de/4V/juOXcta5frjvzXO4/sqEtyytse0g3zZFuWu5cDN0MkLz2qRDD2Ag==",
+      "cpu": [
+        "s390x"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/linux-x64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-x64/-/linux-x64-0.28.1.tgz",
+      "integrity": "sha512-u/anNYF2mmVOEDwLtnQ1wOr3EZ9sTNGLWrsYGYwHWzGA3Si84IOkHXlbWTD1NB+9/1lcnweYKO54uhxZydNzfA==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "linux"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/netbsd-arm64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/netbsd-arm64/-/netbsd-arm64-0.28.1.tgz",
+      "integrity": "sha512-oks0DYbLwWMmaakTsCb+zL4E+aHRVLom9IJZOAthMQEPiQmydXHkziYEsGYRx0uNV/IjEKGAV941JzH02pflqw==",
+      "cpu": [
+        "arm64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "netbsd"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/netbsd-x64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/netbsd-x64/-/netbsd-x64-0.28.1.tgz",
+      "integrity": "sha512-aeL6lAnN89Hz43Mlh1G8ARasbuoYvSITDEx0tHh5b7jJnHcssqgjy9Yx430GDpmCa6OyrKoS0aNRjKundRizGg==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "netbsd"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/openbsd-arm64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/openbsd-arm64/-/openbsd-arm64-0.28.1.tgz",
+      "integrity": "sha512-MEFJe5C3R8pwXdZ5Y21oo6m7ePiS0d9pWucn99O/wvyJZChoIQKrQDxKrGeW8F5+T0okTHesAmDeiHDTIq0V/Q==",
+      "cpu": [
+        "arm64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "openbsd"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/openbsd-x64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/openbsd-x64/-/openbsd-x64-0.28.1.tgz",
+      "integrity": "sha512-i/ZLIOafE0Z8cI/XANJAixoJL/uRAoS2xOA3rb0xN+KK0K177cMAsQYkzHtBrtMXAKuAc7HGgcWiZ/sRC1Nxgw==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "openbsd"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/openharmony-arm64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/openharmony-arm64/-/openharmony-arm64-0.28.1.tgz",
+      "integrity": "sha512-ge+Z7EXFNt2BO1oAMsVpiQ8EwndV9i1xXerAeTIK7AtPs3bKFXQM7nlRxDSIUIMeueR1CNXxqztLzdNeReKBJg==",
+      "cpu": [
+        "arm64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "openharmony"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/sunos-x64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/sunos-x64/-/sunos-x64-0.28.1.tgz",
+      "integrity": "sha512-BEjgtECkL3vY+SaSQ6nzVfiALUeFxpawyp8Jmf5PtYhf1Ug40N1h/hxlhts+f1FvSvarEigdxS3BlSMI2PJLcQ==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "sunos"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/win32-arm64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/win32-arm64/-/win32-arm64-0.28.1.tgz",
+      "integrity": "sha512-lCv9eK/H6ZJWbE7bh2nw54CZ9M2nupBxJcTsdk/QQnWkdSjKGuxmmH8/GWrlT1eMmZfn4dGcCjRte397WqfQXA==",
+      "cpu": [
+        "arm64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "win32"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/win32-ia32": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/win32-ia32/-/win32-ia32-0.28.1.tgz",
+      "integrity": "sha512-zvb/mB2bSCoJOpoCBgYKKpX6YM6mJBlBUVUtVj41DlZJVEB6/0CKlRYxP5wWl1C1ILiCoAU5wZZ4q1P3qeS6Eg==",
+      "cpu": [
+        "ia32"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "win32"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/@esbuild/win32-x64": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/@esbuild/win32-x64/-/win32-x64-0.28.1.tgz",
+      "integrity": "sha512-bm4Mowrv+GXMlpWX++EcXw/iLyd1o3+bJkC2DkWXYVvgZCqD/bSj9ctZeAMC3cIxgjRVR2Dufaiu4YPxr5gW1A==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "win32"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/tsx/node_modules/esbuild": {
+      "version": "0.28.1",
+      "resolved": "https://registry.npmjs.org/esbuild/-/esbuild-0.28.1.tgz",
+      "integrity": "sha512-HrJrvZv5ayxBzPfwphOoNzkzOIIlifzk0KJrGK2c8R4+LKpMtpYLQeUdjnwjWv/LZlkH2laZk+4w78pi99D4Vw==",
+      "dev": true,
+      "hasInstallScript": true,
+      "license": "MIT",
+      "bin": {
+        "esbuild": "bin/esbuild"
+      },
+      "engines": {
+        "node": ">=18"
+      },
+      "optionalDependencies": {
+        "@esbuild/aix-ppc64": "0.28.1",
+        "@esbuild/android-arm": "0.28.1",
+        "@esbuild/android-arm64": "0.28.1",
+        "@esbuild/android-x64": "0.28.1",
+        "@esbuild/darwin-arm64": "0.28.1",
+        "@esbuild/darwin-x64": "0.28.1",
+        "@esbuild/freebsd-arm64": "0.28.1",
+        "@esbuild/freebsd-x64": "0.28.1",
+        "@esbuild/linux-arm": "0.28.1",
+        "@esbuild/linux-arm64": "0.28.1",
+        "@esbuild/linux-ia32": "0.28.1",
+        "@esbuild/linux-loong64": "0.28.1",
+        "@esbuild/linux-mips64el": "0.28.1",
+        "@esbuild/linux-ppc64": "0.28.1",
+        "@esbuild/linux-riscv64": "0.28.1",
+        "@esbuild/linux-s390x": "0.28.1",
+        "@esbuild/linux-x64": "0.28.1",
+        "@esbuild/netbsd-arm64": "0.28.1",
+        "@esbuild/netbsd-x64": "0.28.1",
+        "@esbuild/openbsd-arm64": "0.28.1",
+        "@esbuild/openbsd-x64": "0.28.1",
+        "@esbuild/openharmony-arm64": "0.28.1",
+        "@esbuild/sunos-x64": "0.28.1",
+        "@esbuild/win32-arm64": "0.28.1",
+        "@esbuild/win32-ia32": "0.28.1",
+        "@esbuild/win32-x64": "0.28.1"
+      }
+    },
+    "node_modules/tsx/node_modules/fsevents": {
+      "version": "2.3.3",
+      "resolved": "https://registry.npmjs.org/fsevents/-/fsevents-2.3.3.tgz",
+      "integrity": "sha512-5xoDfX+fL7faATnagmWPpbFtwh/R77WmMMqqHGS65C3vvB0YHrgF+B1YmZ3441tMj5n63k0212XNoJwzlhffQw==",
+      "dev": true,
+      "hasInstallScript": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "darwin"
+      ],
+      "engines": {
+        "node": "^8.16.0 || ^10.6.0 || >=11.0.0"
+      }
+    },
+    "node_modules/type-check": {
+      "version": "0.4.0",
+      "resolved": "https://registry.npmjs.org/type-check/-/type-check-0.4.0.tgz",
+      "integrity": "sha512-XleUoc9uwGXqjWwXaUTZAmzMcFZ5858QA2vvx1Ur5xIcixXIP+8LnFDgRplU30us6teqdlskFfu+ae4K79Ooew==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "prelude-ls": "^1.2.1"
+      },
+      "engines": {
+        "node": ">= 0.8.0"
+      }
+    },
+    "node_modules/typed-array-buffer": {
+      "version": "1.0.3",
+      "resolved": "https://registry.npmjs.org/typed-array-buffer/-/typed-array-buffer-1.0.3.tgz",
+      "integrity": "sha512-nAYYwfY3qnzX30IkA6AQZjVbtK6duGontcQm1WSG1MD94YLqK0515GNApXkoxKOWMusVssAHWLh9SeaoefYFGw==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "call-bound": "^1.0.3",
+        "es-errors": "^1.3.0",
+        "is-typed-array": "^1.1.14"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      }
+    },
+    "node_modules/typed-array-byte-length": {
+      "version": "1.0.3",
+      "resolved": "https://registry.npmjs.org/typed-array-byte-length/-/typed-array-byte-length-1.0.3.tgz",
+      "integrity": "sha512-BaXgOuIxz8n8pIq3e7Atg/7s+DpiYrxn4vdot3w9KbnBhcRQq6o3xemQdIfynqSeXeDrF32x+WvfzmOjPiY9lg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "call-bind": "^1.0.8",
+        "for-each": "^0.3.3",
+        "gopd": "^1.2.0",
+        "has-proto": "^1.2.0",
+        "is-typed-array": "^1.1.14"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/ljharb"
+      }
+    },
+    "node_modules/typed-array-byte-offset": {
+      "version": "1.0.4",
+      "resolved": "https://registry.npmjs.org/typed-array-byte-offset/-/typed-array-byte-offset-1.0.4.tgz",
+      "integrity": "sha512-bTlAFB/FBYMcuX81gbL4OcpH5PmlFHqlCCpAl8AlEzMz5k53oNDvN8p1PNOWLEmI2x4orp3raOFB51tv9X+MFQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "available-typed-arrays": "^1.0.7",
+        "call-bind": "^1.0.8",
+        "for-each": "^0.3.3",
+        "gopd": "^1.2.0",
+        "has-proto": "^1.2.0",
+        "is-typed-array": "^1.1.15",
+        "reflect.getprototypeof": "^1.0.9"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/ljharb"
+      }
+    },
+    "node_modules/typed-array-length": {
+      "version": "1.0.7",
+      "resolved": "https://registry.npmjs.org/typed-array-length/-/typed-array-length-1.0.7.tgz",
+      "integrity": "sha512-3KS2b+kL7fsuk/eJZ7EQdnEmQoaho/r6KUef7hxvltNA5DR8NAUM+8wJMbJyZ4G9/7i3v5zPBIMN5aybAh2/Jg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "call-bind": "^1.0.7",
+        "for-each": "^0.3.3",
+        "gopd": "^1.0.1",
+        "is-typed-array": "^1.1.13",
+        "possible-typed-array-names": "^1.0.0",
+        "reflect.getprototypeof": "^1.0.6"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/ljharb"
+      }
+    },
+    "node_modules/typescript": {
+      "version": "5.9.3",
+      "resolved": "https://registry.npmjs.org/typescript/-/typescript-5.9.3.tgz",
+      "integrity": "sha512-jl1vZzPDinLr9eUt3J/t7V6FgNEw9QjvBPdysz9KfQDD41fQrC2Y4vKQdiaUpFT4bXlb1RHhLpp8wtm6M5TgSw==",
+      "dev": true,
+      "license": "Apache-2.0",
+      "bin": {
+        "tsc": "bin/tsc",
+        "tsserver": "bin/tsserver"
+      },
+      "engines": {
+        "node": ">=14.17"
+      }
+    },
+    "node_modules/typescript-eslint": {
+      "version": "8.59.2",
+      "resolved": "https://registry.npmjs.org/typescript-eslint/-/typescript-eslint-8.59.2.tgz",
+      "integrity": "sha512-pJw051uomb3ZeCzGTpRb8RbEqB5Y4WWet8gl/GcTlU35BSx0PVdZ86/bqkQCyKKuraVQEK7r6kBHQXF+fBhkoQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@typescript-eslint/eslint-plugin": "8.59.2",
+        "@typescript-eslint/parser": "8.59.2",
+        "@typescript-eslint/typescript-estree": "8.59.2",
         "@typescript-eslint/utils": "8.59.2"
       },
       "engines": {
@@ -8502,13 +8991,13 @@
       }
     },
     "node_modules/vite": {
-      "version": "7.3.6",
-      "resolved": "https://registry.npmjs.org/vite/-/vite-7.3.6.tgz",
-      "integrity": "sha512-4XP60spRGjSZFf1qYH+dJIkK2znL3zQfl9KkOV9MkkRR/3Dls0dxaBsQPTloEc5BLXWPL9vsOxopxyKoMmDueg==",
+      "version": "7.3.2",
+      "resolved": "https://registry.npmjs.org/vite/-/vite-7.3.2.tgz",
+      "integrity": "sha512-Bby3NOsna2jsjfLVOHKes8sGwgl4TT0E6vvpYgnAYDIF/tie7MRaFthmKuHx1NSXjiTueXH3do80FMQgvEktRg==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
-        "esbuild": "^0.27.0 || ^0.28.0",
+        "esbuild": "^0.27.0",
         "fdir": "^6.5.0",
         "picomatch": "^4.0.3",
         "postcss": "^8.5.6",
@@ -8617,10 +9106,25 @@
         }
       }
     },
+    "node_modules/vite/node_modules/fsevents": {
+      "version": "2.3.3",
+      "resolved": "https://registry.npmjs.org/fsevents/-/fsevents-2.3.3.tgz",
+      "integrity": "sha512-5xoDfX+fL7faATnagmWPpbFtwh/R77WmMMqqHGS65C3vvB0YHrgF+B1YmZ3441tMj5n63k0212XNoJwzlhffQw==",
+      "dev": true,
+      "hasInstallScript": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "darwin"
+      ],
+      "engines": {
+        "node": "^8.16.0 || ^10.6.0 || >=11.0.0"
+      }
+    },
     "node_modules/vite/node_modules/picomatch": {
-      "version": "4.0.5",
-      "resolved": "https://registry.npmjs.org/picomatch/-/picomatch-4.0.5.tgz",
-      "integrity": "sha512-RvwwcruNjI1ncT5xRakeyS9Lf8lcItv34KD+aif+VH9kduAyfYBipGh12274xtenIPZ119/R9BdTBa8gAwSh0A==",
+      "version": "4.0.4",
+      "resolved": "https://registry.npmjs.org/picomatch/-/picomatch-4.0.4.tgz",
+      "integrity": "sha512-QP88BAKvMam/3NxH6vj2o21R6MjxZUAd6nlwAS/pnGvN9IVLocLHxGYIzFhg6fUQ+5th6P4dv4eW9jX3DSIj7A==",
       "dev": true,
       "license": "MIT",
       "engines": {
@@ -8631,20 +9135,20 @@
       }
     },
     "node_modules/vitest": {
-      "version": "3.2.7",
-      "resolved": "https://registry.npmjs.org/vitest/-/vitest-3.2.7.tgz",
-      "integrity": "sha512-KrxIJ62Fd89gfysR4WotlgZABiz2dqFPgqGzX7s+CwsqLFomRH7777ZcrOD6+WVAh7khPQP41A+BKbpcJFrdEg==",
+      "version": "3.2.4",
+      "resolved": "https://registry.npmjs.org/vitest/-/vitest-3.2.4.tgz",
+      "integrity": "sha512-LUCP5ev3GURDysTWiP47wRRUpLKMOfPh+yKTx3kVIEiu5KOMeqzpnYNsKyOoVrULivR8tLcks4+lga33Whn90A==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
         "@types/chai": "^5.2.2",
-        "@vitest/expect": "3.2.7",
-        "@vitest/mocker": "3.2.7",
-        "@vitest/pretty-format": "^3.2.7",
-        "@vitest/runner": "3.2.7",
-        "@vitest/snapshot": "3.2.7",
-        "@vitest/spy": "3.2.7",
-        "@vitest/utils": "3.2.7",
+        "@vitest/expect": "3.2.4",
+        "@vitest/mocker": "3.2.4",
+        "@vitest/pretty-format": "^3.2.4",
+        "@vitest/runner": "3.2.4",
+        "@vitest/snapshot": "3.2.4",
+        "@vitest/spy": "3.2.4",
+        "@vitest/utils": "3.2.4",
         "chai": "^5.2.0",
         "debug": "^4.4.1",
         "expect-type": "^1.2.1",
@@ -8674,8 +9178,8 @@
         "@edge-runtime/vm": "*",
         "@types/debug": "^4.1.12",
         "@types/node": "^18.0.0 || ^20.0.0 || >=22.0.0",
-        "@vitest/browser": "3.2.7",
-        "@vitest/ui": "3.2.7",
+        "@vitest/browser": "3.2.4",
+        "@vitest/ui": "3.2.4",
         "happy-dom": "*",
         "jsdom": "*"
       },
@@ -8704,9 +9208,9 @@
       }
     },
     "node_modules/vitest/node_modules/picomatch": {
-      "version": "4.0.5",
-      "resolved": "https://registry.npmjs.org/picomatch/-/picomatch-4.0.5.tgz",
-      "integrity": "sha512-RvwwcruNjI1ncT5xRakeyS9Lf8lcItv34KD+aif+VH9kduAyfYBipGh12274xtenIPZ119/R9BdTBa8gAwSh0A==",
+      "version": "4.0.4",
+      "resolved": "https://registry.npmjs.org/picomatch/-/picomatch-4.0.4.tgz",
+      "integrity": "sha512-QP88BAKvMam/3NxH6vj2o21R6MjxZUAd6nlwAS/pnGvN9IVLocLHxGYIzFhg6fUQ+5th6P4dv4eW9jX3DSIj7A==",
       "dev": true,
       "license": "MIT",
       "engines": {
@@ -8739,6 +9243,20 @@
         "node": ">=20"
       }
     },
+    "node_modules/whatwg-encoding": {
+      "version": "3.1.1",
+      "resolved": "https://registry.npmjs.org/whatwg-encoding/-/whatwg-encoding-3.1.1.tgz",
+      "integrity": "sha512-6qN4hJdMwfYBtE3YBTTHhoeuUrDBPZmbQaxWAqSALV/MeEnR5z1xd8UKud2RAkFoPkmB+hli1TZSnyi84xz1vQ==",
+      "deprecated": "Use @exodus/bytes instead for a more spec-conformant and faster implementation",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "iconv-lite": "0.6.3"
+      },
+      "engines": {
+        "node": ">=18"
+      }
+    },
     "node_modules/whatwg-mimetype": {
       "version": "4.0.0",
       "resolved": "https://registry.npmjs.org/whatwg-mimetype/-/whatwg-mimetype-4.0.0.tgz",
@@ -8896,9 +9414,9 @@
       }
     },
     "node_modules/ws": {
-      "version": "8.21.3",
-      "resolved": "https://registry.npmjs.org/ws/-/ws-8.21.3.tgz",
-      "integrity": "sha512-201TZ/kPWxoPr/OKWjquZR1SWKXcvxdH+e1xrx89b3YbmzLMFCLfnaG1HFIgWzJOEWZ7MvpK++odZufgYR50Rw==",
+      "version": "8.20.0",
+      "resolved": "https://registry.npmjs.org/ws/-/ws-8.20.0.tgz",
+      "integrity": "sha512-sAt8BhgNbzCtgGbt2OxmpuryO63ZoDk/sqaB/znQm94T4fCEsy/yV+7CdC1kJhOU9lboAEU7R3kquuycDoibVA==",
       "dev": true,
       "license": "MIT",
       "engines": {
diff --git a/package.json b/package.json
index 55dabbf..fde3f1b 100644
--- a/package.json
+++ b/package.json
@@ -11,7 +11,8 @@
     "typecheck": "tsc --noEmit",
     "content:check": "node --import tsx scripts/validate-content.ts",
     "test": "vitest run",
-    "test:watch": "vitest"
+    "test:watch": "vitest",
+    "test:e2e": "playwright test"
   },
   "dependencies": {
     "next": "16.2.4",
@@ -21,6 +22,7 @@
     "zod": "^4.4.3"
   },
   "devDependencies": {
+    "@playwright/test": "^1.59.1",
     "@tailwindcss/postcss": "^4",
     "@testing-library/jest-dom": "^6.9.1",
     "@testing-library/react": "^16.3.2",
diff --git a/playwright.config.ts b/playwright.config.ts
new file mode 100644
index 0000000..13229fd
--- /dev/null
+++ b/playwright.config.ts
@@ -0,0 +1,34 @@
+import { defineConfig, devices } from "@playwright/test";
+
+export default defineConfig({
+  testDir: "./tests/e2e",
+  timeout: 30_000,
+  expect: {
+    timeout: 5_000,
+  },
+  reporter: [["list"]],
+  outputDir: "test-results",
+  // Next's development compiler can invalidate an in-flight navigation when
+  // both device projects request a cold route at the same time.
+  workers: 1,
+  use: {
+    baseURL: "http://localhost:3100",
+    trace: "on-first-retry",
+  },
+  webServer: {
+    command: "npm run dev",
+    url: "http://localhost:3100",
+    reuseExistingServer: true,
+    timeout: 120_000,
+  },
+  projects: [
+    {
+      name: "chromium",
+      use: { ...devices["Desktop Chrome"] },
+    },
+    {
+      name: "mobile-chrome",
+      use: { ...devices["Pixel 7"] },
+    },
+  ],
+});
diff --git a/tests/e2e/portfolio.spec.ts b/tests/e2e/portfolio.spec.ts
new file mode 100644
index 0000000..27eca3f
--- /dev/null
+++ b/tests/e2e/portfolio.spec.ts
@@ -0,0 +1,489 @@
+import { expect, test, type Locator, type Page } from "@playwright/test";
+
+import contactJson from "../../src/content/contact.json";
+import curationJson from "../../src/content/curation.json";
+import experienceJson from "../../src/content/experience.json";
+import interviewMapJson from "../../src/content/interview-map.json";
+import journeyNarrativeJson from "../../src/content/journey-narrative.json";
+import linksJson from "../../src/content/links.json";
+import presentationJson from "../../src/content/presentation.json";
+import profileJson from "../../src/content/profile.json";
+import projectsJson from "../../src/content/projects.json";
+import resumeJson from "../../src/content/resume.json";
+import siteJson from "../../src/content/site.json";
+import skillsJson from "../../src/content/skills.json";
+import techStackJson from "../../src/content/tech-stack.json";
+
+const designIds = [
+  "design",
+  "classic",
+  "editorial",
+  "brutalist",
+  "cinematic",
+] as const;
+type DesignId = (typeof designIds)[number];
+
+function requireFixture<T>(value: T | undefined, message: string): T {
+  if (value === undefined) {
+    throw new Error(message);
+  }
+
+  return value;
+}
+
+const firstProject = requireFixture(
+  projectsJson.items.find((project) => project.enabled !== false),
+  "The portfolio needs at least one enabled project.",
+);
+const firstProjectTechnology = requireFixture(
+  techStackJson.find((technology) => technology.id === firstProject.stack[0]),
+  "The first project technology must resolve in E2E fixtures.",
+);
+const firstResumeProject = requireFixture(
+  projectsJson.items.find((project) => project.id === resumeJson.projectIds[0]),
+  "The first resume project must resolve in E2E fixtures.",
+);
+const firstInterviewAnswer = interviewMapJson.tracks[0]?.items[0]?.answers[0];
+const firstInterviewProject = requireFixture(
+  projectsJson.items.find(
+    (project) => project.id === firstInterviewAnswer?.projectId,
+  ),
+  "The first interview project must resolve in E2E fixtures.",
+);
+
+const templateLabels = new Map(
+  presentationJson.templates.map((template) => [template.id, template.label]),
+);
+
+const routeDefinitions = [
+  { path: "/", pageId: undefined },
+  { path: "/projects", pageId: "projects" },
+  { path: `/projects/${firstProject.id}`, pageId: "projects" },
+  { path: "/about", pageId: "about" },
+  { path: "/resume", pageId: "resume" },
+  { path: "/contact", pageId: "contact" },
+  { path: "/journey", pageId: "journey" },
+  { path: "/interview-map", pageId: "interviewMap" },
+] as const;
+
+const enabledRoutes = routeDefinitions.filter(
+  ({ pageId }) => !pageId || siteJson.pages?.[pageId] !== false,
+);
+
+const projectsNavigation = siteJson.navigation.find(
+  (item) => item.href === "/projects",
+);
+
+if (!projectsNavigation) {
+  throw new Error("site.json must include a /projects navigation item.");
+}
+
+function withExplicitDesign(path: string, designId: DesignId) {
+  const url = new URL(path, "https://portfolio.test");
+  url.searchParams.set("view", designId);
+  return `${url.pathname}${url.search}${url.hash}`;
+}
+
+function expectedInternalHref(
+  path: string,
+  designId: DesignId,
+  contentDebug = false,
+) {
+  const url = new URL(path, "https://portfolio.test");
+
+  if (designId === presentationJson.defaultHomeTemplate) {
+    url.searchParams.delete("view");
+  } else {
+    url.searchParams.set("view", designId);
+  }
+
+  if (contentDebug) {
+    url.searchParams.set("debug", "content");
+  }
+
+  return `${url.pathname}${url.search}${url.hash}`;
+}
+
+function expectedActiveDesignNavigationHref(path: string, designId: DesignId) {
+  return expectedInternalHref(path, designId);
+}
+
+async function expectDesignRoute(
+  page: Page,
+  path: string,
+  designId: DesignId,
+) {
+  const response = await page.goto(withExplicitDesign(path, designId));
+
+  expect(response?.ok()).toBe(true);
+  await expect(
+    page.locator(`[data-site-design="${designId}"]`),
+  ).toBeVisible();
+  await expect(page.locator("main")).toBeVisible();
+  await expect(page.getByRole("heading", { level: 1 }).first()).toBeVisible();
+  const activeLabel = templateLabels.get(designId);
+
+  if (!activeLabel) {
+    throw new Error(`Missing presentation copy for ${designId}.`);
+  }
+
+  await expect(
+    page.getByLabel(
+      presentationJson.ui.designSwitcherAriaTemplate.replace(
+        "{label}",
+        activeLabel,
+      ),
+    ),
+  ).toBeVisible();
+  expect(
+    await page.evaluate(
+      () => document.documentElement.scrollWidth <= window.innerWidth + 1,
+    ),
+  ).toBe(true);
+}
+
+async function expectExactContent(page: Page, value: string) {
+  await expect(
+    page.locator("main").getByText(value, { exact: true }).first(),
+  ).toBeVisible();
+}
+
+async function expectContainedContent(page: Page, value: string) {
+  await expect(
+    page.locator("main").getByText(value, { exact: false }).first(),
+  ).toBeVisible();
+}
+
+async function expectMinimumTouchTargets(locator: Locator) {
+  const bounds = await locator.evaluateAll((elements) =>
+    elements.map((element) => {
+      const rectangle = element.getBoundingClientRect();
+      return { height: rectangle.height, width: rectangle.width };
+    }),
+  );
+
+  expect(bounds.length).toBeGreaterThan(0);
+  for (const target of bounds) {
+    expect(target.height).toBeGreaterThanOrEqual(44);
+    expect(target.width).toBeGreaterThanOrEqual(44);
+  }
+}
+
+async function expectSharedRouteEvidence(page: Page, path: string) {
+  if (path === "/") {
+    await expectExactContent(page, profileJson.headline);
+    await expectExactContent(page, firstProject.title);
+    return;
+  }
+
+  if (path === "/projects") {
+    await expectExactContent(page, firstProject.title);
+    const firstGroup = projectsJson.groups.find(
+      (group) => group.id === firstProject.groupId,
+    );
+    if (firstGroup) {
+      await expectContainedContent(page, firstGroup.label);
+    }
+    return;
+  }
+
+  if (path.startsWith("/projects/")) {
+    await expectExactContent(page, firstProject.title);
+    await expectContainedContent(page, firstProjectTechnology.label);
+    await expectExactContent(page, firstProject.highlights[0]);
+
+    const projectImage = page.locator("main img").first();
+    await expect(projectImage).toBeVisible();
+    expect(
+      await projectImage.evaluate((image) => {
+        const element = image as HTMLImageElement;
+        const bounds = element.getBoundingClientRect();
+
+        return (
+          element.complete &&
+          element.naturalWidth > 0 &&
+          element.naturalHeight > 0 &&
+          bounds.width > 0 &&
+          bounds.height > 0
+        );
+      }),
+    ).toBe(true);
+    return;
+  }
+
+  if (path === "/about") {
+    await expectExactContent(page, skillsJson.focusAreas[0].title);
+    await expectExactContent(page, experienceJson[0].title);
+    await expect(
+      page.locator("main").getByAltText(profileJson.photo.alt),
+    ).toBeVisible();
+
+    if (siteJson.pages.curation) {
+      await expectExactContent(page, curationJson.nextReview.title);
+    }
+    return;
+  }
+
+  if (path === "/resume") {
+    await expectContainedContent(page, profileJson.availability);
+    await expectContainedContent(page, resumeJson.summary[0]);
+    await expectExactContent(page, resumeJson.notes[0]);
+    await expectExactContent(page, firstResumeProject.title);
+    await expectExactContent(page, resumeJson.training[0].name);
+    await expectExactContent(page, experienceJson[0].title);
+    return;
+  }
+
+  if (path === "/contact") {
+    await expectExactContent(page, contactJson.availability);
+    await expectExactContent(page, contactJson.notes[0]);
+    const firstPreferredLink = contactJson.preferred[0];
+    const firstPreferredLinkLabel = linksJson.find(
+      (link) => link.id === firstPreferredLink,
+    )?.label;
+    if (firstPreferredLinkLabel) {
+      await expectContainedContent(page, firstPreferredLinkLabel);
+    }
+    return;
+  }
+
+  if (path === "/journey") {
+    await expectExactContent(page, journeyNarrativeJson.milestones[0].title);
+    await expectContainedContent(page, journeyNarrativeJson.milestones[0].state);
+    await expectExactContent(page, journeyNarrativeJson.currentPosition.title);
+    await expectExactContent(page, journeyNarrativeJson.currentPosition.body);
+    return;
+  }
+
+  if (path === "/interview-map") {
+    await expectContainedContent(page, interviewMapJson.referenceRepo.label);
+    await expectContainedContent(page, interviewMapJson.tracks[0].items[0].label);
+    await expectContainedContent(page, firstInterviewProject.title);
+    await expectExactContent(page, interviewMapJson.gaps.items[0]);
+  }
+}
+
+for (const designId of designIds) {
+  test(`${designId}: renders every enabled route on shared content`, async ({
+    page,
+  }) => {
+    test.slow();
+
+    for (const { path } of enabledRoutes) {
+      await expectDesignRoute(page, path, designId);
+      await expectSharedRouteEvidence(page, path);
+    }
+  });
+
+}
+
+test("design switching preserves the current route and content debug query", async ({
+  page,
+}) => {
+  test.slow();
+  const hydrationErrors: string[] = [];
+
+  page.on("console", (message) => {
+    if (
+      message.type() === "error" &&
+      /hydration|server rendered HTML|did not match/i.test(message.text())
+    ) {
+      hydrationErrors.push(message.text());
+    }
+  });
+
+  for (const [index, sourceDesign] of designIds.entries()) {
+    const targetDesign = designIds[(index + 1) % designIds.length];
+    await expectDesignRoute(
+      page,
+      "/projects?debug=content",
+      sourceDesign,
+    );
+
+    await page
+      .locator('summary[aria-label^="Change site design"]')
+      .click();
+    const designNavigation = page.getByRole("navigation", {
+      name: "Site design",
+    });
+    await expect(designNavigation).toBeVisible();
+
+    const expectedHref = expectedInternalHref(
+      "/projects",
+      targetDesign,
+      true,
+    );
+    await designNavigation.locator(`a[href="${expectedHref}"]`).click();
+
+    await expect(page).toHaveURL(
+      new RegExp(`${expectedHref.replace(/[?&]/g, "\\$&")}$`),
+    );
+    await expect(
+      page.locator(`[data-site-design="${targetDesign}"]`),
+    ).toBeVisible();
+  }
+
+  expect(hydrationErrors).toEqual([]);
+});
+
+test("an invalid design query falls back to editorial", async ({ page }) => {
+  const response = await page.goto("/?view=not-a-design");
+
+  expect(response?.ok()).toBe(true);
+  await expect(page.locator('[data-site-design="editorial"]')).toBeVisible();
+  await expect(
+    page.getByLabel(
+      presentationJson.ui.designSwitcherAriaTemplate.replace(
+        "{label}",
+        templateLabels.get("editorial") ?? "editorial",
+      ),
+    ),
+  ).toBeVisible();
+});
+
+test("reduced motion disables sustained presentation animation", async ({
+  page,
+}) => {
+  test.slow();
+  await page.emulateMedia({ reducedMotion: "reduce" });
+
+  for (const designId of designIds) {
+    await expectDesignRoute(page, "/", designId);
+    const motionState = await page.evaluate(() => {
+      const durationToMilliseconds = (value: string) => {
+        const numeric = Number.parseFloat(value);
+        return value.trim().endsWith("ms") ? numeric : numeric * 1000;
+      };
+
+      let maximumDuration = 0;
+      let hasInfiniteAnimation = false;
+
+      for (const element of document.querySelectorAll("*")) {
+        for (const pseudoElement of [null, "::before", "::after"] as const) {
+          const style = getComputedStyle(element, pseudoElement);
+          const durations = [
+            ...style.animationDuration.split(","),
+            ...style.transitionDuration.split(","),
+          ].map(durationToMilliseconds);
+          maximumDuration = Math.max(maximumDuration, ...durations);
+          hasInfiniteAnimation ||= style.animationIterationCount
+            .split(",")
+            .some((count) => count.trim() === "infinite");
+        }
+      }
+
+      return {
+        hasInfiniteAnimation,
+        maximumDuration,
+        runningInfiniteAnimations: document
+          .getAnimations()
+          .some(
+            (animation) =>
+              animation.playState === "running" &&
+              animation.effect?.getTiming().iterations === Infinity,
+          ),
+        scrollBehavior: getComputedStyle(document.documentElement)
+          .scrollBehavior,
+      };
+    });
+
+    expect(motionState.maximumDuration).toBeLessThanOrEqual(1);
+    expect(motionState.hasInfiniteAnimation).toBe(false);
+    expect(motionState.runningInfiniteAnimations).toBe(false);
+    expect(motionState.scrollBehavior).toBe("auto");
+  }
+});
+
+test("mobile navigation reaches projects without losing the active design", async ({
+  page,
+}, testInfo) => {
+  test.skip(!testInfo.project.name.includes("mobile"));
+  test.slow();
+
+  for (const designId of designIds) {
+    await expectDesignRoute(page, "/", designId);
+    const mobileNavigation = page.locator(
+      `nav[aria-label="${presentationJson.ui.mobileNavigationAriaLabel}"]`,
+    );
+    const menu = page
+      .locator("details")
+      .filter({ has: mobileNavigation })
+      .locator(":scope > summary");
+    const switcher = page.getByLabel(
+      presentationJson.ui.designSwitcherAriaTemplate.replace(
+        "{label}",
+        templateLabels.get(designId) ?? designId,
+      ),
+    );
+
+    await expect(menu).toBeVisible();
+    await expect(switcher).toBeVisible();
+    await expectMinimumTouchTargets(menu);
+    await expectMinimumTouchTargets(switcher);
+
+    await switcher.click();
+    const designNavigation = page.getByRole("navigation", {
+      name: presentationJson.ui.designNavigationAriaLabel,
+    });
+    await expect(designNavigation).toBeVisible();
+    await expectMinimumTouchTargets(designNavigation.getByRole("link"));
+    const closeDesignSheet = designNavigation.getByRole("button", {
+      name: presentationJson.ui.designSwitcherCloseLabel,
+    });
+    await expect(closeDesignSheet).toBeVisible();
+    await expectMinimumTouchTargets(closeDesignSheet);
+    const sheetBounds = await designNavigation.boundingBox();
+    expect(sheetBounds).not.toBeNull();
+    const sheetPosition = await page.evaluate(
+      ({ bottom, left, right }) => ({
+        bottomGap: Math.abs(window.innerHeight - bottom),
+        left,
+        rightOverflow: Math.max(0, right - window.innerWidth),
+      }),
+      {
+        bottom: (sheetBounds?.y ?? 0) + (sheetBounds?.height ?? 0),
+        left: sheetBounds?.x ?? 0,
+        right: (sheetBounds?.x ?? 0) + (sheetBounds?.width ?? 0),
+      },
+    );
+    expect(sheetPosition.bottomGap).toBeLessThanOrEqual(2);
+    expect(sheetPosition.left).toBeGreaterThanOrEqual(0);
+    expect(sheetPosition.rightOverflow).toBeLessThanOrEqual(1);
+    await closeDesignSheet.click();
+    await expect(designNavigation).toBeHidden();
+    await expect(switcher).toBeFocused();
+
+    await menu.click();
+    await expect(mobileNavigation).toBeVisible();
+    const expectedHref = expectedActiveDesignNavigationHref(
+      "/projects",
+      designId,
+    );
+    const projectsLink = mobileNavigation
+      .locator(`a[href="${expectedHref}"]`)
+      .first();
+
+    await expect(projectsLink).toBeVisible();
+    await expect(projectsLink).toContainText(projectsNavigation.label);
+    await expectMinimumTouchTargets(mobileNavigation.getByRole("link"));
+    await menu.focus();
+    await page.keyboard.press("Tab");
+    const firstMobileLink = mobileNavigation.getByRole("link").first();
+    await expect(firstMobileLink).toBeFocused();
+    expect(
+      await firstMobileLink.evaluate((element) => {
+        const style = getComputedStyle(element);
+        return (
+          style.outlineStyle !== "none" && Number.parseFloat(style.outlineWidth) > 0
+        ) || style.boxShadow !== "none";
+      }),
+    ).toBe(true);
+    await projectsLink.click();
+    await expect(page).toHaveURL(
+      new RegExp(`${expectedHref.replace(/[?&]/g, "\\$&")}$`),
+    );
+    await expect(
+      page.locator(`[data-site-design="${designId}"]`),
+    ).toBeVisible();
+  }
+});


