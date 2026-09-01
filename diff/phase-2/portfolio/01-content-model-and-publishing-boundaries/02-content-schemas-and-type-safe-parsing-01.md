# 콘텐츠 스키마와 타입 안전 파싱

## `build(content): runtime 콘텐츠 검증 의존성 추가`

diff --git a/package-lock.json b/package-lock.json
index b5e64b4..d40cbd7 100644
--- a/package-lock.json
+++ b/package-lock.json
@@ -11,7 +11,8 @@
         "next": "16.2.4",
         "react": "19.2.4",
         "react-dom": "19.2.4",
-        "simple-icons": "^16.18.1"
+        "simple-icons": "^16.18.1",
+        "zod": "^4.4.3"
       },
       "devDependencies": {
         "@tailwindcss/postcss": "^4",
@@ -21,6 +22,7 @@
         "eslint": "^9",
         "eslint-config-next": "16.2.4",
         "tailwindcss": "^4",
+        "tsx": "^4.23.0",
         "typescript": "^5"
       }
     },
@@ -310,6 +312,448 @@
         "tslib": "^2.4.0"
       }
     },
+    "node_modules/@esbuild/aix-ppc64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/aix-ppc64/-/aix-ppc64-0.28.2.tgz",
+      "integrity": "sha512-XExcO+dvLKvVtNTibSTBej1NCAbaGhWn9Ww1ZPx80qsahhPFe/8jgWP0IchNe0F3HwkU7n8ejhH8bjonqht8mQ==",
+      "cpu": [
+        "ppc64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "aix"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/@esbuild/android-arm": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/android-arm/-/android-arm-0.28.2.tgz",
+      "integrity": "sha512-kXXoiPVVGQcnIYGOeaovwOURpniDBpSq4A03qkQ+BMQqtGG6HYap3xne9C1O1yo4TR3qxlCX5IqqmX6fFo2Lqg==",
+      "cpu": [
+        "arm"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "android"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/@esbuild/android-arm64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/android-arm64/-/android-arm64-0.28.2.tgz",
+      "integrity": "sha512-5YfKeeI8qWfBZIX+u2xZC3Zlb3Os/gLS2sbEKM+I4ZOcsWmHS2WLysCcQZDAFRslDUU5Oiq44gf6PYN1vGwG5A==",
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
+        "node": ">=18"
+      }
+    },
+    "node_modules/@esbuild/android-x64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/android-x64/-/android-x64-0.28.2.tgz",
+      "integrity": "sha512-O387ite7SzUyCcy3JQX4P4bLtEA7bLLkx+esve5JHnyYfNTxcVpXZo9jhdB0lTKN44gztELTdU7nS8Nr16Fs1Q==",
+      "cpu": [
+        "x64"
+      ],
+      "dev": true,
+      "license": "MIT",
+      "optional": true,
+      "os": [
+        "android"
+      ],
+      "engines": {
+        "node": ">=18"
+      }
+    },
+    "node_modules/@esbuild/darwin-arm64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/darwin-arm64/-/darwin-arm64-0.28.2.tgz",
+      "integrity": "sha512-n4KqkOQrraxHJcgjM1RvwbigfQKIKJVpM7xp+KsxiyUSrRdIXnt73VhrPAx0fV44hgfmIVKjxMN9J1t5jySVkw==",
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
+        "node": ">=18"
+      }
+    },
+    "node_modules/@esbuild/darwin-x64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/darwin-x64/-/darwin-x64-0.28.2.tgz",
+      "integrity": "sha512-uq6suIWYP37qzGddBKPw5QEQPi6HiLGsO7UmkpfyaYNQ3D+rN6w6WfwH+nuqcGXWvawGwxOEroO4YGnFh95azw==",
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
+        "node": ">=18"
+      }
+    },
+    "node_modules/@esbuild/freebsd-arm64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/freebsd-arm64/-/freebsd-arm64-0.28.2.tgz",
+      "integrity": "sha512-n+I0BTSRIoy+d6RPKnEVwql5UwBJolytvY4mAOIEJorKlqgPII8ix6slVVrfZ5Tnj7glIZvloylbB/EJPMWEXw==",
+      "cpu": [
+        "arm64"
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
+    "node_modules/@esbuild/freebsd-x64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/freebsd-x64/-/freebsd-x64-0.28.2.tgz",
+      "integrity": "sha512-78XJTJkvPs0kz2w61301PJjXl4g7q3JqiYMZ/M/yVI73EHBrCRTgkhu9oqG7vPqq+a/yadEW8aD+agKlk5xrmg==",
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
+    "node_modules/@esbuild/linux-arm": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-arm/-/linux-arm-0.28.2.tgz",
+      "integrity": "sha512-XlDnu2q5yoqems+xay6wSAcg9DDD7K9RLKZEBOMZm3ckNpJBvOX20tSfby8KfrrhINDyv9V2YVZKY/SpoGJI8w==",
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
+    "node_modules/@esbuild/linux-arm64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-arm64/-/linux-arm64-0.28.2.tgz",
+      "integrity": "sha512-pW4AC0P3it8c7do9MVM4p51FzHzdM/TZrerurgRcHJ2WTa1VQ1CIq18xncfpBJw4ojkiZZrKW2yIBWBP92j6Ug==",
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
+    "node_modules/@esbuild/linux-ia32": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-ia32/-/linux-ia32-0.28.2.tgz",
+      "integrity": "sha512-CYbnj78HsIeA+DhgUKgFCfvNsTHFhMMrinUrMZpDXJXKN8T3XViTZ/+wtHeVxEWY8ewSzTFN+nRmSwO2tZaLUQ==",
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
+    "node_modules/@esbuild/linux-loong64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-loong64/-/linux-loong64-0.28.2.tgz",
+      "integrity": "sha512-buwkd8nsph4R+ajRvw0qM5Hja/TXQow3ptzWO2EbG/cqcIkHloRrdlBtQlshyYGTNFvfkfJ5tpPLVkY4DtsPfQ==",
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
+    "node_modules/@esbuild/linux-mips64el": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-mips64el/-/linux-mips64el-0.28.2.tgz",
+      "integrity": "sha512-ZVykbDyk7519VwiNb9Lcj9m8XM6v5V9uKPvrEMkkEedVewf+0itkhahp4HDpgERXhwLRpWFypsGbG/J8s0QjJA==",
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
+    "node_modules/@esbuild/linux-ppc64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-ppc64/-/linux-ppc64-0.28.2.tgz",
+      "integrity": "sha512-CAXl+Dtd9UUuJd8pKKdwh6MLm3MUMiqMPmhZ3tTSXPqfyQ3vDl6R5hZdZ/kYojK4ofXtdfSv1tFq8XzWx3heNQ==",
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
+    "node_modules/@esbuild/linux-riscv64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-riscv64/-/linux-riscv64-0.28.2.tgz",
+      "integrity": "sha512-GeXCej4IQtU1B+QlDV8W/RRvbzI3O/Stss+/bCXv4lZls5WGRtu2a+3JkA3i4qIUlMXpcHebWpF8AkJhATowuA==",
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
+    "node_modules/@esbuild/linux-s390x": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-s390x/-/linux-s390x-0.28.2.tgz",
+      "integrity": "sha512-3H1weTYZPxt/WOhByszQZybS9w5lKzUn1FDMsgEChbHWQwHYQQRfBxgCcZvPhjHfKyJjIievvMmEUawJrdY9Dg==",
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
+    "node_modules/@esbuild/linux-x64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/linux-x64/-/linux-x64-0.28.2.tgz",
+      "integrity": "sha512-4xTZr1FUmSoQW4XIWmit3tzQrUTZM+N3P0XV8xROKYF50XfI7xeO90+1bZvNwxIufQ9hDQVRJH5YhgPVF8A/HQ==",
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
+    "node_modules/@esbuild/netbsd-arm64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/netbsd-arm64/-/netbsd-arm64-0.28.2.tgz",
+      "integrity": "sha512-sSATRjPeDBg3pdgHoQfoYBob11Kk1FGa9lui5RIHZCoCkJa9QKlvl3/vKz2usCmYYjs7ymJR/2Nnsqe+Hjt5nw==",
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
+    "node_modules/@esbuild/netbsd-x64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/netbsd-x64/-/netbsd-x64-0.28.2.tgz",
+      "integrity": "sha512-lqnzCV+mM0gIADaKihiCg6ifgfU2L3h5E33rNQBN1Y4MaVGnzryzmvvf7UHxprpQdE8hpqLolJ9Rl+SkIRDpyw==",
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
+    "node_modules/@esbuild/openbsd-arm64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/openbsd-arm64/-/openbsd-arm64-0.28.2.tgz",
+      "integrity": "sha512-AL2qJILH7lNjrDmCQDvdxMfAUIv8KMNZOvrwAQ8i8//ntL9FflhOyMJ8OZSMBb8/AWXe3/5v5S20y3zCoZWKoQ==",
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
+    "node_modules/@esbuild/openbsd-x64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/openbsd-x64/-/openbsd-x64-0.28.2.tgz",
+      "integrity": "sha512-QtiuPytchRyC4rwUKhexJdQKvDuZ6hWloi3igqPQNUJCS1/v9EiO3UTOXR6A3FoMo4fnAKbWJdqaIwhOzh8qEw==",
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
+    "node_modules/@esbuild/openharmony-arm64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/openharmony-arm64/-/openharmony-arm64-0.28.2.tgz",
+      "integrity": "sha512-WkhYDmpTjLvGlScA1rwjRUmhl4k8oXR3cIbtqWmELgU/dFeHHlEllxDvdWcNJV9rbzCexB5vz8gtNewWLgCT7Q==",
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
+    "node_modules/@esbuild/sunos-x64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/sunos-x64/-/sunos-x64-0.28.2.tgz",
+      "integrity": "sha512-GPMSkTOtMnv2U2F8gxe4Io6qmVs+YKyp832Etqqxr0hFngmXQ3rzwytelm3GIn7T4VviRUlf3sOgBOiTdvaf7g==",
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
+    "node_modules/@esbuild/win32-arm64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/win32-arm64/-/win32-arm64-0.28.2.tgz",
+      "integrity": "sha512-PIhhEkE9uPBleRBrQEJpUn7MBnibZzbGzYWPmY3x+YoVg/95zbjB4CxPPOQ8l5tYYM4mMaCthF8/1DIfBQQyWQ==",
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
+    "node_modules/@esbuild/win32-ia32": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/win32-ia32/-/win32-ia32-0.28.2.tgz",
+      "integrity": "sha512-YmJbfTlvU7Sdn9BB+4PRES4oB6pxgS37MAONj+hBr/cpXS1aBPKXxNnDbu+QCWPj0o9dgyxeq79g6c5P8KeuYA==",
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
+    "node_modules/@esbuild/win32-x64": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/@esbuild/win32-x64/-/win32-x64-0.28.2.tgz",
+      "integrity": "sha512-5ebpxr3nWMzrL/rnUI755Jkuee0bHL/Gq0WTF9lvcpv73wAp5eu8MfBUgWK9bhWvZjj7yX8etf/8tI8Ney695g==",
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
     "node_modules/@eslint-community/eslint-utils": {
       "version": "4.9.1",
       "resolved": "https://registry.npmjs.org/@eslint-community/eslint-utils/-/eslint-utils-4.9.1.tgz",
@@ -3041,6 +3485,48 @@
         "url": "https://github.com/sponsors/ljharb"
       }
     },
+    "node_modules/esbuild": {
+      "version": "0.28.2",
+      "resolved": "https://registry.npmjs.org/esbuild/-/esbuild-0.28.2.tgz",
+      "integrity": "sha512-HKVLS8dvII+xoKW9kmqxbRKrnWEXfJJr/FZhhJmiqIB0e053QNYFqOBouTMO/k5sID4MvCiUCvv8b9M4h32wIA==",
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
+        "@esbuild/aix-ppc64": "0.28.2",
+        "@esbuild/android-arm": "0.28.2",
+        "@esbuild/android-arm64": "0.28.2",
+        "@esbuild/android-x64": "0.28.2",
+        "@esbuild/darwin-arm64": "0.28.2",
+        "@esbuild/darwin-x64": "0.28.2",
+        "@esbuild/freebsd-arm64": "0.28.2",
+        "@esbuild/freebsd-x64": "0.28.2",
+        "@esbuild/linux-arm": "0.28.2",
+        "@esbuild/linux-arm64": "0.28.2",
+        "@esbuild/linux-ia32": "0.28.2",
+        "@esbuild/linux-loong64": "0.28.2",
+        "@esbuild/linux-mips64el": "0.28.2",
+        "@esbuild/linux-ppc64": "0.28.2",
+        "@esbuild/linux-riscv64": "0.28.2",
+        "@esbuild/linux-s390x": "0.28.2",
+        "@esbuild/linux-x64": "0.28.2",
+        "@esbuild/netbsd-arm64": "0.28.2",
+        "@esbuild/netbsd-x64": "0.28.2",
+        "@esbuild/openbsd-arm64": "0.28.2",
+        "@esbuild/openbsd-x64": "0.28.2",
+        "@esbuild/openharmony-arm64": "0.28.2",
+        "@esbuild/sunos-x64": "0.28.2",
+        "@esbuild/win32-arm64": "0.28.2",
+        "@esbuild/win32-ia32": "0.28.2",
+        "@esbuild/win32-x64": "0.28.2"
+      }
+    },
     "node_modules/escalade": {
       "version": "3.2.0",
       "resolved": "https://registry.npmjs.org/escalade/-/escalade-3.2.0.tgz",
@@ -3611,6 +4097,21 @@
         "url": "https://github.com/sponsors/ljharb"
       }
     },
+    "node_modules/fsevents": {
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
     "node_modules/function-bind": {
       "version": "1.1.2",
       "resolved": "https://registry.npmjs.org/function-bind/-/function-bind-1.1.2.tgz",
@@ -6212,6 +6713,25 @@
       "integrity": "sha512-oJFu94HQb+KVduSUQL7wnpmqnfmLsOA/nAh6b6EH0wCEoK0/mPeXU6c3wKDV83MkOuHPRHtSXKKU99IBazS/2w==",
       "license": "0BSD"
     },
+    "node_modules/tsx": {
+      "version": "4.23.12",
+      "resolved": "https://registry.npmjs.org/tsx/-/tsx-4.23.12.tgz",
+      "integrity": "sha512-FDf4L4sYzKtzWYhU/Xm0AQFdTjdIxNo9ElTf2mxXM6k8YMHXzYUe4yODVaXP4V9uMFbVg8c0qyBccK2OOxb45Q==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "esbuild": "~0.28.0"
+      },
+      "bin": {
+        "tsx": "dist/cli.mjs"
+      },
+      "engines": {
+        "node": ">=18.0.0"
+      },
+      "optionalDependencies": {
+        "fsevents": "~2.3.3"
+      }
+    },
     "node_modules/type-check": {
       "version": "0.4.0",
       "resolved": "https://registry.npmjs.org/type-check/-/type-check-0.4.0.tgz",
@@ -6582,7 +7102,6 @@
       "version": "4.4.3",
       "resolved": "https://registry.npmjs.org/zod/-/zod-4.4.3.tgz",
       "integrity": "sha512-ytENFjIJFl2UwYglde2jchW2Hwm4GJFLDiSXWdTrJQBIN9Fcyp7n4DhxJEiWNAJMV1/BqWfW/kkg71UDcHJyTQ==",
-      "dev": true,
       "license": "MIT",
       "funding": {
         "url": "https://github.com/sponsors/colinhacks"
diff --git a/package.json b/package.json
index 7d062de..ce9c6c4 100644
--- a/package.json
+++ b/package.json
@@ -13,7 +13,8 @@
     "next": "16.2.4",
     "react": "19.2.4",
     "react-dom": "19.2.4",
-    "simple-icons": "^16.18.1"
+    "simple-icons": "^16.18.1",
+    "zod": "^4.4.3"
   },
   "devDependencies": {
     "@tailwindcss/postcss": "^4",
@@ -23,6 +24,7 @@
     "eslint": "^9",
     "eslint-config-next": "16.2.4",
     "tailwindcss": "^4",
+    "tsx": "^4.23.0",
     "typescript": "^5"
   }
 }


## `feat(content): 콘텐츠 경로와 기본 식별자 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
new file mode 100644
index 0000000..eedc836
--- /dev/null
+++ b/src/lib/content-schema.ts
@@ -0,0 +1,32 @@
+import { z } from "zod";
+
+const nonEmptyString = z.string().trim().min(1);
+const contentId = nonEmptyString.regex(
+  /^[a-z0-9]+(?:[a-zA-Z0-9-]*[a-zA-Z0-9])?$/,
+  "Use a stable alphanumeric id (hyphens are allowed).",
+);
+const color = z.string().regex(/^#[0-9a-fA-F]{6}$/, "Use a six-digit hex color.");
+
+export const contentHrefSchema = nonEmptyString.refine(
+  (href) =>
+    href.startsWith("/") ||
+    href.startsWith("#") ||
+    href.startsWith("https://") ||
+    href.startsWith("http://") ||
+    href.startsWith("mailto:") ||
+    href.startsWith("tel:"),
+  "Use a root-relative path or an http(s), mailto, or tel URL.",
+);
+
+export const contentAssetPathSchema = nonEmptyString.refine(
+  (assetPath) =>
+    assetPath.startsWith("/content/") || assetPath.startsWith("/template/"),
+  "Local assets must live under public/content or public/template.",
+);
+
+export const navigationItemSchema = z
+  .object({
+    label: nonEmptyString,
+    href: contentHrefSchema,
+  })
+  .strict();


## `feat(content): 사이트와 프로필 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index eedc836..430f5b9 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -30,3 +30,60 @@ export const navigationItemSchema = z
     href: contentHrefSchema,
   })
   .strict();
+
+export const siteContentSchema = z
+  .object({
+    title: nonEmptyString,
+    description: nonEmptyString,
+    language: nonEmptyString,
+    brand: nonEmptyString,
+    socialImage: contentAssetPathSchema.optional(),
+    pages: z
+      .object({
+        projects: z.boolean(),
+        about: z.boolean(),
+        resume: z.boolean(),
+        contact: z.boolean(),
+        journey: z.boolean(),
+        interviewMap: z.boolean(),
+        curation: z.boolean(),
+      })
+      .strict()
+      .optional(),
+    navigation: z.array(navigationItemSchema),
+    footer: z
+      .object({
+        note: nonEmptyString,
+        copyright: nonEmptyString,
+      })
+      .strict(),
+  })
+  .passthrough();
+
+export const profileContentSchema = z
+  .object({
+    name: nonEmptyString,
+    koreanName: z.string(),
+    handle: nonEmptyString,
+    role: nonEmptyString,
+    headline: nonEmptyString,
+    summary: nonEmptyString,
+    location: nonEmptyString,
+    availability: nonEmptyString,
+    photo: z
+      .object({
+        src: contentAssetPathSchema,
+        alt: nonEmptyString,
+      })
+      .strict()
+      .optional(),
+    principles: z.array(
+      z
+        .object({
+          title: nonEmptyString,
+          body: nonEmptyString,
+        })
+        .strict(),
+    ),
+  })
+  .strict();


