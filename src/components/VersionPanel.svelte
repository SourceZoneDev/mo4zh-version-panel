<script lang="ts">
    import RotateCcwIcon from "@lucide/svelte/icons/rotate-ccw";
    import { toast } from "svelte-sonner";
    import * as Alert from "$lib/components/ui/alert/index.js";
    import { Button } from "$lib/components/ui/button/index.js";
    import * as Card from "$lib/components/ui/card/index.js";
    import * as Dialog from "$lib/components/ui/dialog/index.js";
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

    const emptyVersion = (): VersionInfo => ({
        version: "",
        gameVersion: "",
        message: "",
        url: "",
    });

    let loaded = $state(false);
    let loadDescription = $state("");
    let release = $state(emptyVersion());
    let test = $state(emptyVersion());
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

            release = { ...data.release };
            test = { ...data.test };
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

        let hasError = false;
        if (!release.version) {
            fieldErrors.releaseVersion = "不能为空";
            hasError = true;
        }
        if (!release.gameVersion) {
            fieldErrors.releaseGameVersion = "不能为空";
            hasError = true;
        }
        if (!test.version) {
            fieldErrors.testVersion = "不能为空";
            hasError = true;
        }
        if (!test.gameVersion) {
            fieldErrors.testGameVersion = "不能为空";
            hasError = true;
        }

        if (hasError) {
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

        try {
            const res = await fetch(new URL("/api/v1/version", API_BASE_URL), {
                method: "PUT",
                headers: {
                    "Content-Type": "application/json",
                    Authorization: `Bearer ${accessToken}`,
                },
                body: JSON.stringify({ release, test }),
                signal: abortController.signal,
            });

            if (res.ok) {
                isDialogOpen = false;
                dialogError = "";
                initialRelease = { ...release };
                initialTest = { ...test };
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

    function resetField(target: "release" | "test", field: keyof VersionInfo) {
        if (target === "release") {
            release[field] = initialRelease[field];
            if (field === "version" && release.version)
                fieldErrors.releaseVersion = "";
            if (field === "gameVersion" && release.gameVersion)
                fieldErrors.releaseGameVersion = "";
            return;
        }

        test[field] = initialTest[field];
        if (field === "version" && test.version) fieldErrors.testVersion = "";
        if (field === "gameVersion" && test.gameVersion)
            fieldErrors.testGameVersion = "";
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
    }

    function errorFor(
        target: "release" | "test",
        field: "version" | "gameVersion",
    ) {
        if (target === "release" && field === "version")
            return fieldErrors.releaseVersion;
        if (target === "release" && field === "gameVersion")
            return fieldErrors.releaseGameVersion;
        if (target === "test" && field === "version")
            return fieldErrors.testVersion;
        return fieldErrors.testGameVersion;
    }
</script>

{#snippet FieldLabel(
    target: "release" | "test",
    field: keyof VersionInfo,
    label: string,
    error = "",
)}
    <span
        class="mb-1.5 flex min-h-5 items-center gap-1 text-sm font-medium text-foreground"
    >
        {label}
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
        {#if error}
            <span class="ml-1 text-xs font-normal text-destructive"
                >{error}</span
            >
        {/if}
    </span>
{/snippet}

{#snippet VersionForm(target: "release" | "test")}
    {@const data = target === "release" ? release : test}
    <div
        class="animate-in fade-in slide-in-from-bottom-[5px] flex flex-col gap-4 pt-4 duration-300 ease-out"
    >
        <div class="grid gap-4 sm:grid-cols-2">
            <label class="block">
                {@render FieldLabel(
                    target,
                    "version",
                    "版本号",
                    errorFor(target, "version"),
                )}
                <Input
                    bind:value={data.version}
                    placeholder={target === "release"
                        ? "必填，例如：1.0"
                        : "必填，例如：1.1"}
                    aria-invalid={Boolean(errorFor(target, "version"))}
                    oninput={() => clearRequiredError(target, "version")}
                />
            </label>

            <label class="block">
                {@render FieldLabel(
                    target,
                    "gameVersion",
                    "对应游戏版本号",
                    errorFor(target, "gameVersion"),
                )}
                <Input
                    bind:value={data.gameVersion}
                    placeholder={target === "release"
                        ? "必填，例如：1.0.0"
                        : "必填，例如：1.0.1"}
                    aria-invalid={Boolean(errorFor(target, "gameVersion"))}
                    oninput={() => clearRequiredError(target, "gameVersion")}
                />
            </label>
        </div>

        <label class="block">
            {@render FieldLabel(target, "message", "更新信息")}
            <Textarea bind:value={data.message} class="min-h-28 resize-y" />
        </label>

        <label class="block">
            {@render FieldLabel(target, "url", "下载地址")}
            <Input
                bind:value={data.url}
                placeholder={target === "release"
                    ? "例如：https://www.game.dev/blog/0/"
                    : "例如：https://www.game.dev/blog/1/"}
            />
        </label>
    </div>
{/snippet}

<main
    class="box-border flex min-h-screen items-center justify-center bg-[oklch(0.9642_0_0)] p-4 sm:p-8"
>
    <Card.Root class="w-full max-w-[600px] p-6 sm:p-8">
        {#if loadDescription}
            <Alert.Root variant="destructive" class="mb-1">
                <Alert.Description>{loadDescription}</Alert.Description>
            </Alert.Root>
        {/if}

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
                <Button
                    disabled={!isModified || saving || !loaded}
                    onclick={save}>保存</Button
                >
            </div>
        </div>
    </Card.Root>

    <Dialog.Root bind:open={isDialogOpen}>
        <Dialog.Content showCloseButton={false} class="max-w-[332px]">
            <Dialog.Header>
                <Dialog.Title>请输入密钥</Dialog.Title>
                <Dialog.Description
                    >保存版本信息需要访问密钥。</Dialog.Description
                >
            </Dialog.Header>

            {#if dialogError}
                <p class="text-sm text-destructive">{dialogError}</p>
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
