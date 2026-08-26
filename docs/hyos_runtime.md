# HyperOS Rust Runtime API

Zygisk Next provides a native API for modules that target applications running in Xiaomi's
HyperOS Rust Runtime. This environment is independent of ART and does not provide JNI or
Java objects.

## Registration

Add the module library to `zn_modules.txt`:

```text
path=/system_ext/bin/hyos_spawner lib/arm64-v8a/libexample.so
```

Export the regular `zn_module` entry. During `onModuleLoaded`, query the current runtime and
register a `ZygiskNextHyosModule` when the Hyos Runtime API is available:

```cpp
#include "zygisk_next_api.h"

static void app_specialized(const ZnHyosAppSpecializeArgs *args) {
    // Application-specific initialization.
}

static const ZygiskNextHyosModule hyos_module = {
    .target_api_version = ZYGISK_NEXT_HYOS_API_VERSION,
    .onAppSpecialized = app_specialized,
};

static void on_module_loaded(void *, const ZygiskNextAPI *api) {
    const ZygiskNextRuntime *runtime = api->getRuntime();
    if (runtime == nullptr || runtime->type != ZN_RUNTIME_HYOS ||
        runtime->api_version < ZYGISK_NEXT_HYOS_API_VERSION) {
        return;
    }

    if (runtime->registerModule(&hyos_module) != ZN_SUCCESS) {
        // Registration failed.
    }
}

extern "C" __attribute__((visibility("default"))) ZygiskNextModule zn_module = {
    .target_api_version = ZYGISK_NEXT_API_VERSION,
    .onModuleLoaded = on_module_loaded,
};
```

`getRuntime()` returns `nullptr` when the current process does not expose a supported runtime.
Always check `type` and `api_version` before passing the corresponding runtime-specific module
structure to `registerModule`.

Call `registerModule` during `onModuleLoaded`. The runtime copies the supplied module structure
before returning. The `ZygiskNextAPI` and `ZygiskNextRuntime` objects remain valid for the lifetime
of the process.

## Specialization callback

`onAppSpecialized` is called once in each successfully specialized application process, after its
uid, gid, supplementary groups, and SELinux application context have been applied. The callback
runs before the runtime continues application execution.

`ZnHyosAppSpecializeArgs` contains:

- `process_name`: non-null process name.
- `package_name`: non-null package name.
- `se_info`: non-null SELinux seinfo value used for application context selection.

The argument structure and strings are read-only and valid only for the duration of the callback.
Modifying them does not affect process specialization.

The module is inherited by every child selected by the manifest entry. Validate the process or
package name inside `onAppSpecialized` before performing application-specific work.

Because the callback runs in a post-fork child, avoid operations that depend on inherited locks,
starting threads, and allocator-heavy work. The standard `ZygiskNextAPI`, including companion
connections, remains available from the callback.
