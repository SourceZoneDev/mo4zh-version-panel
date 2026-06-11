<script lang="ts">
    import RotateCcwIcon from "@lucide/svelte/icons/rotate-ccw";
    import { get, writable } from "svelte/store";
    import { toast } from "svelte-sonner";
    import * as Alert from "$lib/components/ui/alert/index.js";
    import { Button } from "$lib/components/ui/button/index.js";
    import * as Card from "$lib/components/ui/card/index.js";
    import * as Dialog from "$lib/components/ui/dialog/index.js";
    import * as Form from "$lib/components/ui/form/index.js";
    import { Input } from "$lib/components/ui/input/index.js";
    import { Toaster } from "$lib/components/ui/sonner/index.js";
    import { Spinner } from "$lib/components/ui/spinner/index.js";
    import * as Tabs from "$lib/components/ui/tabs/index.js";
    import { Textarea } from "$lib/components/ui/textarea/index.js";

    const API_BASE_URL = "http://103.45.162.207:47434";

    type VersionInfo = {
        version: string;
        gameVersion: string;
        message: string;
        url: string;
    };

    type VersionPayload = {
        release: VersionInfo;
        test: VersionInfo;
    };

    type VersionFormData = {
        release: VersionInfo;
        test: VersionInfo;
    };

    type FormControlProps = {
        id?: string;
        name?: string;
        "aria-describedby"?: string;
        "aria-required"?: boolean | "true" | "false";
        "data-fs-control"?: string;
        "data-fs-error"?: string;
    };

    const emptyVersion = (): VersionInfo => ({
        version: "",
        gameVersion: "",
        message: "",
        url: "",
    });

    const emptyForm = (): VersionFormData => ({
        release: emptyVersion(),
        test: emptyVersion(),
    });

    type VersionFormErrors = {
        release?: Partial<Record<keyof VersionInfo, string[] | undefined>>;
        test?: Partial<Record<keyof VersionInfo, string[] | undefined>>;
    };

    const formData = writable<VersionFormData>(emptyForm());
    const formErrors = writable<VersionFormErrors>({});
    const formConstraints = writable({});
    const formTainted = writable({});

    const form = {
        form: formData,
        errors: {
            subscribe: formErrors.subscribe,
            set: (value: VersionFormErrors) => formErrors.set(value),
            update: (
                updater: (value: VersionFormErrors) => VersionFormErrors,
            ) => formErrors.update(updater),
            clear: () => formErrors.set({}),
        },
        constraints: formConstraints,
        tainted: formTainted,
    };

    let loaded = $state(false);
    let loadDescription = $state("");
    let initialRelease = $state(emptyVersion());
    let initialTest = $state(emptyVersion());
    let saving = $state(false);
    let isDialogOpen = $state(false);
    let accessToken = $state("");
    let dialogError = $state("");
    let abortController: AbortController | null = null;
    let fieldErrors = $state({
        releaseVersion: "",
        releaseGameVersion: "",
        testVersion: "",
        testGameVersion: "",
    });

    let currentForm = $derived($formData);
    let release = $derived(currentForm.release);
    let test = $derived(currentForm.test);
    let isModified = $derived(
        release.version !== initialRelease.version ||
            release.gameVersion !== initialRelease.gameVersion ||
            release.message !== initialRelease.message ||
            release.url !== initialRelease.url ||
            test.version !== initialTest.version ||
            test.gameVersion !== initialTest.gameVersion ||
            test.message !== initialTest.message ||
            test.url !== initialTest.url,
    );

    $effect(() => {
        void loadVersion();

        return () => {
            abortController?.abort();
        };
    });

    async function parseJsonResponse(res: Response) {
        try {
            return await res.json();
        } catch (error) {
            return {
                title: error instanceof Error ? error.message : String(error),
            };
        }
    }

    async function loadVersion() {
        try {
            const res = await fetch(new URL("/api/v1/version", API_BASE_URL));
            const data = (await parseJsonResponse(
                res,
            )) as Partial<VersionPayload> & { title?: string };

            if (!res.ok || !data.release || !data.test) {
                loadDescription = `版本获取失败，请刷新页面（${res.statusText || data.title || "响应格式错误"}）`;
                return;
            }

            formData.set({
                release: { ...data.release },
                test: { ...data.test },
            });
            initialRelease = { ...data.release };
            initialTest = { ...data.test };
        } catch (error) {
            const message =
                error instanceof Error ? error.message : String(error);
            loadDescription = `版本请求失败，请刷新页面（${message}）`;
        } finally {
            loaded = true;
        }
    }

    function closeDialog() {
        abortController?.abort();
        abortController = null;
        isDialogOpen = false;
        dialogError = "";
    }

    function save() {
        if (saving || !isModified) return;

        fieldErrors.releaseVersion = "";
        fieldErrors.releaseGameVersion = "";
        fieldErrors.testVersion = "";
        fieldErrors.testGameVersion = "";
        form.errors.clear();

        const payload = get(formData);
        let hasError = false;
        if (!payload.release.version) {
            fieldErrors.releaseVersion = "不能为空";
            hasError = true;
        }
        if (!payload.release.gameVersion) {
            fieldErrors.releaseGameVersion = "不能为空";
            hasError = true;
        }
        if (!payload.test.version) {
            fieldErrors.testVersion = "不能为空";
            hasError = true;
        }
        if (!payload.test.gameVersion) {
            fieldErrors.testGameVersion = "不能为空";
            hasError = true;
        }

        if (hasError) {
            form.errors.set({
                release: {
                    version: fieldErrors.releaseVersion
                        ? [fieldErrors.releaseVersion]
                        : undefined,
                    gameVersion: fieldErrors.releaseGameVersion
                        ? [fieldErrors.releaseGameVersion]
                        : undefined,
                },
                test: {
                    version: fieldErrors.testVersion
                        ? [fieldErrors.testVersion]
                        : undefined,
                    gameVersion: fieldErrors.testGameVersion
                        ? [fieldErrors.testGameVersion]
                        : undefined,
                },
            });
            toast.warning("字段不能为空", {
                description: "请填写正式版和测试版的版本号、对应游戏版本号。",
            });
            return;
        }

        dialogError = "";
        isDialogOpen = true;
    }

    async function confirmSave() {
        if (!accessToken || saving) return;

        saving = true;
        loadDescription = "";
        dialogError = "";
        abortController = new AbortController();

        const payload = get(formData);

        try {
            const res = await fetch(new URL("/api/v1/version", API_BASE_URL), {
                method: "PUT",
                headers: {
                    "Content-Type": "application/json",
                    Authorization: `Bearer ${accessToken}`,
                },
                body: JSON.stringify(payload),
                signal: abortController.signal,
            });

            if (res.ok) {
                isDialogOpen = false;
                dialogError = "";
                initialRelease = { ...payload.release };
                initialTest = { ...payload.test };
                toast.success("版本更新完成");
            } else if (res.status === 401) {
                dialogError = "密钥错误，请重试";
            } else {
                isDialogOpen = false;
                const data = await parseJsonResponse(res);
                toast.error("版本更新失败，请重试", {
                    description: res.statusText || data.title || "未知错误",
                });
            }
        } catch (error) {
            if (error instanceof DOMException && error.name === "AbortError")
                return;

            isDialogOpen = false;
            const message =
                error instanceof Error ? error.message : String(error);
            toast.error("版本更新失败，请重试", {
                description: message,
            });
        } finally {
            saving = false;
            abortController = null;
        }
    }

    function setField(
        target: "release" | "test",
        field: keyof VersionInfo,
        value: string,
    ) {
        formData.update((data) => ({
            ...data,
            [target]: {
                ...data[target],
                [field]: value,
            },
        }));
    }

    function resetField(target: "release" | "test", field: keyof VersionInfo) {
        if (target === "release") {
            setField("release", field, initialRelease[field]);
            if (field === "version" && initialRelease.version)
                fieldErrors.releaseVersion = "";
            if (field === "gameVersion" && initialRelease.gameVersion)
                fieldErrors.releaseGameVersion = "";
            form.errors.clear();
            return;
        }

        setField("test", field, initialTest[field]);
        if (field === "version" && initialTest.version)
            fieldErrors.testVersion = "";
        if (field === "gameVersion" && initialTest.gameVersion)
            fieldErrors.testGameVersion = "";
        form.errors.clear();
    }

    function isFieldModified(
        target: "release" | "test",
        field: keyof VersionInfo,
    ) {
        return target === "release"
            ? release[field] !== initialRelease[field]
            : test[field] !== initialTest[field];
    }

    function clearRequiredError(
        target: "release" | "test",
        field: "version" | "gameVersion",
    ) {
        if (target === "release" && field === "version" && release.version)
            fieldErrors.releaseVersion = "";
        if (
            target === "release" &&
            field === "gameVersion" &&
            release.gameVersion
        )
            fieldErrors.releaseGameVersion = "";
        if (target === "test" && field === "version" && test.version)
            fieldErrors.testVersion = "";
        if (target === "test" && field === "gameVersion" && test.gameVersion)
            fieldErrors.testGameVersion = "";
        form.errors.clear();
    }
</script>

{#snippet ResetButton(
    target: "release" | "test",
    field: keyof VersionInfo,
    label: string,
)}
    {#if isFieldModified(target, field)}
        <button
            type="button"
            class="inline-flex rounded-full p-0.5 text-muted-foreground transition hover:rotate-12 hover:bg-muted hover:text-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring"
            title="还原"
            onclick={() => resetField(target, field)}
        >
            <RotateCcwIcon class="size-3.5" />
            <span class="sr-only">还原{label}</span>
        </button>
    {/if}
{/snippet}

{#snippet TextField(
    target: "release" | "test",
    field: "version" | "gameVersion" | "url",
    name:
        | "release.version"
        | "release.gameVersion"
        | "release.url"
        | "test.version"
        | "test.gameVersion"
        | "test.url",
    label: string,
    placeholder: string,
)}
    <Form.Field {form} {name}>
        <Form.Control>
            {#snippet children({ props }: { props: FormControlProps })}
                <Form.Label>
                    {label}
                    {@render ResetButton(target, field, label)}
                </Form.Label>
                <Input
                    {...props}
                    value={target === "release" ? release[field] : test[field]}
                    {placeholder}
                    aria-invalid={field !== "url" &&
                        Boolean($formErrors?.[target]?.[field])}
                    oninput={(event) => {
                        setField(target, field, event.currentTarget.value);
                        if (field !== "url") clearRequiredError(target, field);
                    }}
                />
            {/snippet}
        </Form.Control>
        <Form.FieldErrors />
    </Form.Field>
{/snippet}

{#snippet TextareaField(
    target: "release" | "test",
    name: "release.message" | "test.message",
    label: string,
)}
    <Form.Field {form} {name}>
        <Form.Control>
            {#snippet children({ props }: { props: FormControlProps })}
                <Form.Label>
                    {label}
                    {@render ResetButton(target, "message", label)}
                </Form.Label>
                <Textarea
                    {...props}
                    value={target === "release"
                        ? release.message
                        : test.message}
                    class="min-h-28 resize-y"
                    oninput={(event) =>
                        setField(target, "message", event.currentTarget.value)}
                />
            {/snippet}
        </Form.Control>
        <Form.FieldErrors />
    </Form.Field>
{/snippet}

{#snippet VersionForm(target: "release" | "test")}
    <div
        class="animate-in fade-in slide-in-from-bottom-[5px] flex flex-col gap-4 pt-4 duration-300 ease-out"
    >
        <div class="grid gap-4 sm:grid-cols-2">
            {@render TextField(
                target,
                "version",
                target === "release" ? "release.version" : "test.version",
                "版本号",
                target === "release" ? "必填，例如：1.0" : "必填，例如：1.1",
            )}

            {@render TextField(
                target,
                "gameVersion",
                target === "release"
                    ? "release.gameVersion"
                    : "test.gameVersion",
                "对应游戏版本号",
                target === "release"
                    ? "必填，例如：1.0.0"
                    : "必填，例如：1.0.1",
            )}
        </div>

        {@render TextareaField(
            target,
            target === "release" ? "release.message" : "test.message",
            "更新信息",
        )}

        {@render TextField(
            target,
            "url",
            target === "release" ? "release.url" : "test.url",
            "下载地址",
            target === "release"
                ? "例如：https://www.game.dev/blog/0/"
                : "例如：https://www.game.dev/blog/1/",
        )}
    </div>
{/snippet}

<main
    class="box-border flex min-h-screen items-center justify-center bg-[oklch(0.9642_0_0)] p-4 sm:p-8"
>
    <Card.Root class="w-full max-w-150 p-6 sm:p-8">
        {#if loadDescription}
            <Alert.Root variant="destructive" class="mb-1">
                <Alert.Description>{loadDescription}</Alert.Description>
            </Alert.Root>
        {/if}

        <form
            onsubmit={(event) => {
                event.preventDefault();
                save();
            }}
        >
            <div class="relative mb-1">
                <Tabs.Root value="release" class="w-full">
                    <Tabs.List>
                        <Tabs.Trigger value="release">正式版</Tabs.Trigger>
                        <Tabs.Trigger value="test">测试版</Tabs.Trigger>
                    </Tabs.List>

                    <Tabs.Content value="release">
                        {@render VersionForm("release")}
                    </Tabs.Content>
                    <Tabs.Content value="test">
                        {@render VersionForm("test")}
                    </Tabs.Content>
                </Tabs.Root>

                {#if !loaded}
                    <div
                        class="absolute inset-0 z-10 flex items-center justify-center rounded-2xl bg-background/50 backdrop-blur-[3.33px]"
                    >
                        <Spinner class="size-6" aria-label="正在加载版本信息" />
                    </div>
                {/if}
            </div>

            <div
                class="mt-5 flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between"
            >
                <div class="text-xs text-muted-foreground">
                    本项目采用
                    <a
                        href="https://www.gnu.org/licenses/agpl.html"
                        target="_blank"
                        rel="noopener noreferrer"
                        class="border-b border-dashed border-muted-foreground text-inherit no-underline hover:border-solid"
                    >
                        AGPLv3
                    </a>
                    授权
                    <a
                        href="https://codeberg.org/SourceZoneDev/mo4-version-panel"
                        target="_blank"
                        rel="noopener noreferrer"
                        class="ml-1 border-b border-dashed border-muted-foreground text-inherit no-underline hover:border-solid"
                    >
                        源代码
                    </a>
                </div>

                <div class="flex items-center justify-end gap-4">
                    <Form.Button disabled={!isModified || saving || !loaded}
                        >保存</Form.Button
                    >
                </div>
            </div>
        </form>
    </Card.Root>

    <Dialog.Root bind:open={isDialogOpen}>
        <Dialog.Content showCloseButton={false} class="max-w-83">
            <Dialog.Header>
                <Dialog.Title>请输入密钥</Dialog.Title>
                <Dialog.Description
                    >保存版本信息需要访问密钥。</Dialog.Description
                >
            </Dialog.Header>

            {#if dialogError}
                <Alert.Root variant="destructive" class="mb-1">
                    <Alert.Description>{dialogError}</Alert.Description>
                </Alert.Root>
            {/if}

            <Input
                type="password"
                placeholder="密钥"
                bind:value={accessToken}
                oninput={() => {
                    if (accessToken) dialogError = "";
                }}
            />

            <Dialog.Footer class="items-center">
                {#if saving}
                    <Spinner class="mr-auto size-5" aria-label="正在保存" />
                {/if}
                <Button variant="outline" onclick={closeDialog}>取消</Button>
                <Button disabled={!accessToken || saving} onclick={confirmSave}
                    >确定</Button
                >
            </Dialog.Footer>
        </Dialog.Content>
    </Dialog.Root>

    <Toaster richColors position="top-center" />
</main>
