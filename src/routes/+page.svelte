<script lang="ts">
  import { onMount } from "svelte";
  import { ArrowClockwiseFilled } from "fluentui-icons-svelte";

  const API_BASE_URL = "http://103.45.162.207:47434";

  let loaded = $state(false);
  let loadDescription = $state("");

  let release = $state({
    version: "",
    gameVersion: "",
    message: "",
    url: "",
  });

  let test = $state({
    version: "",
    gameVersion: "",
    message: "",
    url: "",
  });

  let initialRelease = $state({
    version: "",
    gameVersion: "",
    message: "",
    url: "",
  });

  let initialTest = $state({
    version: "",
    gameVersion: "",
    message: "",
    url: "",
  });

  let saving = $state(false);
  let saveDescription = $state("");
  let statusFading = $state(false);
  let statusTimeout: number;

  let isDialogOpen = $state(false);
  let isDialogClosing = $state(false);
  let accessToken = $state("");
  let dialogError = $state("");
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
    test.url !== initialTest.url
  );

  onMount(async () => {
    const {
      provideFluentDesignSystem,
      fluentButton,
      fluentCard,
      fluentDialog,
      fluentProgressRing,
      fluentTab,
      fluentTabs,
      fluentTabPanel,
      fluentTextArea,
      fluentTextField,
    } = await import("@fluentui/web-components");

    provideFluentDesignSystem().register(
      fluentButton(),
      fluentCard(),
      fluentDialog(),
      fluentProgressRing(),
      fluentTab(),
      fluentTabs(),
      fluentTabPanel(),
      fluentTextArea(),
      fluentTextField(),
    );

    try {
      const res = await fetch(new URL("/api/v1/version", API_BASE_URL));
      let data = null;
      try {
        data = await res.json();
      } catch (error) {
        const message = error instanceof Error ? error.message : String(error);
        loadDescription = `版本请求失败: ${res.statusText || message}`;
        return;
      }

      if (!res.ok) {
        loadDescription = `版本获取失败: ${res.statusText || data.title}`;
        return;
      }

      release = data.release;
      test = data.test;

      initialRelease = { ...data.release };
      initialTest = { ...data.test };
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      loadDescription = `版本请求失败: ${message}`;
    } finally {
      loaded = true;
    }
  });

  function showStatus(msg: string) {
    clearTimeout(statusTimeout);
    statusFading = false;

    saveDescription = msg;
    statusTimeout = setTimeout(() => {
      statusFading = true;
      setTimeout(() => {
        saveDescription = "";
        statusFading = false;
      }, 500);
    }, 3000);
  }

  function closeDialog() {
    isDialogClosing = true;
    setTimeout(() => {
      isDialogOpen = false;
      isDialogClosing = false;
      dialogError = "";
    }, 300);
  }

  function save() {
    if (saving) {
      return;
    }

    if (!isModified) {
      return;
    }

    fieldErrors.releaseVersion = "";
    fieldErrors.releaseGameVersion = "";
    fieldErrors.testVersion = "";
    fieldErrors.testGameVersion = "";

    let hasError = false;

    if (!release.version) {
      fieldErrors.releaseVersion = "不能为空";
      hasError = true;
    }
    if (!test.version) {
      fieldErrors.testVersion = "不能为空";
      hasError = true;
    }

    if (!release.gameVersion) {
      fieldErrors.releaseGameVersion = "不能为空";
      hasError = true;
    }
    if (!test.gameVersion) {
      fieldErrors.testGameVersion = "不能为空";
      hasError = true;
    }

    if (hasError) {
      showStatus("字段不能为空");
      return;
    }

    dialogError = "";
    isDialogOpen = true;
    isDialogClosing = false;
  }

  async function confirmSave() {
    if (!accessToken) {
      dialogError = "密钥不能为空";
      return;
    }

    closeDialog();
    saving = true;
    loadDescription = "";

    const payload = {
      release: release,
      test: test,
    };

    try {
      const res = await fetch(new URL("/api/v1/version", API_BASE_URL), {
        method: "PUT",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${accessToken}`,
        },
        body: JSON.stringify(payload),
      });

      if (res.ok) {
        initialRelease = { ...release };
        initialTest = { ...test };

        showStatus("版本已更新");
      } else {
        let data = null;
        try {
          data = await res.json();
        } catch (error) {
          const message =
            error instanceof Error ? error.message : String(error);
          showStatus(`版本更新失败：${res.statusText || message}`);
          return;
        }

        showStatus(`版本更新失败：${res.statusText || data.title}`);
      }
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      showStatus(`版本更新失败：${message}`);
    } finally {
      saving = false;
    }
  }
</script>

<main>
  {#if loaded}
    <fluent-card class="container">
      {#if loadDescription}
        <div class="load-error">
          {loadDescription}
        </div>
      {/if}
      <fluent-tabs>
        <fluent-tab id="release-tab">正式版</fluent-tab>
        <fluent-tab id="test-tab">测试版</fluent-tab>

        <fluent-tab-panel id="release-panel">
          <div class="form-grid">
            <fluent-text-field
              placeholder="必填，例如：1.0"
              value={release.version}
              oninput={(e: any) => {
                release.version = (e.target as HTMLInputElement).value;
                if (release.version) fieldErrors.releaseVersion = "";
              }}
            >
              版本号
              {#if release.version !== initialRelease.version}
                <span
                  class="reset-icon"
                  title="还原"
                  role="button"
                  tabindex="0"
                  onclick={(e) => {
                    e.stopPropagation();
                    release.version = initialRelease.version;
                    if (release.version) fieldErrors.releaseVersion = "";
                  }}
                  onkeydown={(e) => {
                    if (e.key === "Enter" || e.key === " ") {
                      e.preventDefault();
                      e.stopPropagation();
                      release.version = initialRelease.version;
                      if (release.version) fieldErrors.releaseVersion = "";
                    }
                  }}
                >
                  <ArrowClockwiseFilled />
                </span>
              {/if}
              {#if fieldErrors.releaseVersion}
                <span class="error-inline">
                  {fieldErrors.releaseVersion}
                </span>
              {/if}
            </fluent-text-field>

            <fluent-text-field
              placeholder="必填，例如：1.0.0"
              value={release.gameVersion}
              oninput={(e: any) => {
                release.gameVersion = (e.target as HTMLInputElement).value;
                if (release.gameVersion) fieldErrors.releaseGameVersion = "";
              }}
            >
              对应游戏版本号
              {#if release.gameVersion !== initialRelease.gameVersion}
                <span
                  class="reset-icon"
                  title="还原"
                  role="button"
                  tabindex="0"
                  onclick={(e) => {
                    e.stopPropagation();
                    release.gameVersion = initialRelease.gameVersion;
                    if (release.gameVersion)
                      fieldErrors.releaseGameVersion = "";
                  }}
                  onkeydown={(e) => {
                    if (e.key === "Enter" || e.key === " ") {
                      e.preventDefault();
                      e.stopPropagation();
                      release.gameVersion = initialRelease.gameVersion;
                      if (release.gameVersion)
                        fieldErrors.releaseGameVersion = "";
                    }
                  }}
                >
                  <ArrowClockwiseFilled />
                </span>
              {/if}
              {#if fieldErrors.releaseGameVersion}
                <span class="error-inline">
                  {fieldErrors.releaseGameVersion}
                </span>
              {/if}
            </fluent-text-field>

            <fluent-text-area
              resize="vertical"
              value={release.message}
              oninput={(e: any) =>
                (release.message = (e.target as HTMLInputElement).value)}
            >
              更新信息
              {#if release.message !== initialRelease.message}
                <span
                  class="reset-icon"
                  title="还原"
                  role="button"
                  tabindex="0"
                  onclick={(e) => {
                    e.stopPropagation();
                    release.message = initialRelease.message;
                  }}
                  onkeydown={(e) => {
                    if (e.key === "Enter" || e.key === " ") {
                      e.preventDefault();
                      e.stopPropagation();
                      release.message = initialRelease.message;
                    }
                  }}
                >
                  <ArrowClockwiseFilled />
                </span>
              {/if}
            </fluent-text-area>

            <fluent-text-field
              placeholder="例如：https://www.game.dev/blog/0/"
              value={release.url}
              oninput={(e: any) =>
                (release.url = (e.target as HTMLInputElement).value)}
            >
              下载地址
              {#if release.url !== initialRelease.url}
                <span
                  class="reset-icon"
                  title="还原"
                  role="button"
                  tabindex="0"
                  onclick={(e) => {
                    e.stopPropagation();
                    release.url = initialRelease.url;
                  }}
                  onkeydown={(e) => {
                    if (e.key === "Enter" || e.key === " ") {
                      e.preventDefault();
                      e.stopPropagation();
                      release.url = initialRelease.url;
                    }
                  }}
                >
                  <ArrowClockwiseFilled />
                </span>
              {/if}
            </fluent-text-field>
          </div>
        </fluent-tab-panel>

        <fluent-tab-panel id="test-panel">
          <div class="form-grid">
            <fluent-text-field
              placeholder="必填，例如：1.1"
              value={test.version}
              oninput={(e: any) => {
                test.version = (e.target as HTMLInputElement).value;
                if (test.version) fieldErrors.testVersion = "";
              }}
            >
              版本号
              {#if test.version !== initialTest.version}
                <span
                  class="reset-icon"
                  title="还原"
                  role="button"
                  tabindex="0"
                  onclick={(e) => {
                    e.stopPropagation();
                    test.version = initialTest.version;
                    if (test.version) fieldErrors.testVersion = "";
                  }}
                  onkeydown={(e) => {
                    if (e.key === "Enter" || e.key === " ") {
                      e.preventDefault();
                      e.stopPropagation();
                      test.version = initialTest.version;
                      if (test.version) fieldErrors.testVersion = "";
                    }
                  }}
                >
                  <ArrowClockwiseFilled />
                </span>
              {/if}
              {#if fieldErrors.testVersion}
                <span class="error-inline">
                  {fieldErrors.testVersion}
                </span>
              {/if}
            </fluent-text-field>

            <fluent-text-field
              placeholder="必填，例如：1.0.1"
              value={test.gameVersion}
              oninput={(e: any) => {
                test.gameVersion = (e.target as HTMLInputElement).value;
                if (test.gameVersion) fieldErrors.testGameVersion = "";
              }}
            >
              对应游戏版本号
              {#if test.gameVersion !== initialTest.gameVersion}
                <span
                  class="reset-icon"
                  title="还原"
                  role="button"
                  tabindex="0"
                  onclick={(e) => {
                    e.stopPropagation();
                    test.gameVersion = initialTest.gameVersion;
                    if (test.gameVersion) fieldErrors.testGameVersion = "";
                  }}
                  onkeydown={(e) => {
                    if (e.key === "Enter" || e.key === " ") {
                      e.preventDefault();
                      e.stopPropagation();
                      test.gameVersion = initialTest.gameVersion;
                      if (test.gameVersion) fieldErrors.testGameVersion = "";
                    }
                  }}
                >
                  <ArrowClockwiseFilled />
                </span>
              {/if}
              {#if fieldErrors.testGameVersion}
                <span class="error-inline">
                  {fieldErrors.testGameVersion}
                </span>
              {/if}
            </fluent-text-field>

            <fluent-text-area
              resize="vertical"
              value={test.message}
              oninput={(e: any) =>
                (test.message = (e.target as HTMLInputElement).value)}
            >
              更新信息
              {#if test.message !== initialTest.message}
                <span
                  class="reset-icon"
                  title="还原"
                  role="button"
                  tabindex="0"
                  onclick={(e) => {
                    e.stopPropagation();
                    test.message = initialTest.message;
                  }}
                  onkeydown={(e) => {
                    if (e.key === "Enter" || e.key === " ") {
                      e.preventDefault();
                      e.stopPropagation();
                      test.message = initialTest.message;
                    }
                  }}
                >
                  <ArrowClockwiseFilled />
                </span>
              {/if}
            </fluent-text-area>

            <fluent-text-field
              placeholder="例如：https://www.game.dev/blog/1/"
              value={test.url}
              oninput={(e: any) =>
                (test.url = (e.target as HTMLInputElement).value)}
            >
              下载地址
              {#if test.url !== initialTest.url}
                <span
                  class="reset-icon"
                  title="还原"
                  role="button"
                  tabindex="0"
                  onclick={(e) => {
                    e.stopPropagation();
                    test.url = initialTest.url;
                  }}
                  onkeydown={(e) => {
                    if (e.key === "Enter" || e.key === " ") {
                      e.preventDefault();
                      e.stopPropagation();
                      test.url = initialTest.url;
                    }
                  }}
                >
                  <ArrowClockwiseFilled />
                </span>
              {/if}
            </fluent-text-field>
          </div>
        </fluent-tab-panel>
      </fluent-tabs>

      <div class="actions">
        {#if saving}
          <fluent-progress-ring class="save-progress"></fluent-progress-ring>
        {:else if saveDescription}
          <span class="status-message {statusFading ? 'fade-out' : ''} ">
            {saveDescription}
          </span>
        {/if}
        <fluent-button
          appearance="accent"
          onclick={save}
          role="button"
          tabindex="0"
          disabled={saving || !isModified}
          onkeydown={(e: KeyboardEvent) => {
            if (e.key === "Enter" || e.key === " ") {
              e.preventDefault();
              save();
            }
          }}
        >
          保存
        </fluent-button>
      </div>
    </fluent-card>
  {:else}
    <fluent-progress-ring></fluent-progress-ring>
  {/if}

  <fluent-dialog
    hidden={!isDialogOpen}
    class={isDialogClosing ? "dialog-closing" : ""}
  >
    <div class="dialog-content">
      <h3 style="margin-top: 0;">请输入密钥</h3>
      {#if dialogError}
        <div class="dialog-error-text">
          {dialogError}
        </div>
      {/if}
      <fluent-text-field
        type="password"
        placeholder="密钥"
        value={accessToken}
        oninput={(e: any) => {
          accessToken = (e.target as HTMLInputElement).value;
          if (accessToken) dialogError = "";
        }}
        style="width: 100%; margin-bottom: 1rem;"
      ></fluent-text-field>
      <div class="dialog-actions">
        <fluent-button
          onclick={closeDialog}
          role="button"
          tabindex="0"
          onkeydown={(e: KeyboardEvent) => {
            if (e.key === "Enter" || e.key === " ") {
              e.preventDefault();
              closeDialog();
            }
          }}>取消</fluent-button
        >
        <fluent-button
          appearance="accent"
          onclick={confirmSave}
          role="button"
          tabindex="0"
          onkeydown={(e: KeyboardEvent) => {
            if (e.key === "Enter" || e.key === " ") {
              e.preventDefault();
              confirmSave();
            }
          }}>确定</fluent-button
        >
      </div>
    </div>
  </fluent-dialog>
</main>

<style>
  :global(body) {
    font-family: sans-serif;
    background-color: #f3f3f3;
    margin: 0;
    padding: 0;
  }

  main {
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
    padding: 2rem;
    box-sizing: border-box;
  }

  .container {
    padding: 2rem;
    width: 100%;
    max-width: 600px;
  }

  fluent-tabs {
    margin-bottom: 1.5rem;
  }

  fluent-tab-panel {
    animation: tab-fade-in 0.33s ease-out;
  }

  @keyframes tab-fade-in {
    from {
      opacity: 0;
      transform: translateY(5px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  .form-grid {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    padding-top: 1rem;
  }

  fluent-text-field,
  fluent-text-area {
    width: 100%;
  }

  .actions {
    display: flex;
    justify-content: flex-end;
    align-items: center;
    gap: 1rem;
  }

  .save-progress {
    width: 20px;
    height: 20px;
  }

  .load-error {
    color: #d13438;
    background-color: #fde7e9;
    padding: 0.75rem;
    border-radius: 4px;
    margin-bottom: 1rem;
    border: 1px solid #f9ced2;
  }

  .status-message {
    font-size: 0.875rem;
    color: #666;
    animation: fade-in 0.33s ease-out;
    opacity: 1;
    transition: opacity 0.5s ease-out;
  }

  .status-message.fade-out {
    opacity: 0;
  }

  fluent-dialog {
    --dialog-width: auto;
    --dialog-height: auto;
    animation: dialog-fade-in 0.16s ease-out;
  }

  fluent-dialog.dialog-closing {
    animation: dialog-fade-out 0.16s ease-out forwards;
  }

  .dialog-content {
    padding: 1.5rem;
    width: 300px;
    display: flex;
    flex-direction: column;
    background-color: #ffffff;
    border-radius: 4px;
  }

  .dialog-error-text {
    color: #d13438;
    font-size: 0.875rem;
    margin-bottom: 0.5rem;
  }

  .error-inline {
    color: #d13438;
    font-size: 0.875rem;
    margin-left: 0.5rem;
  }

  .reset-icon {
    display: inline-flex;
    vertical-align: text-bottom;
    transition: transform 0.3s ease-out;
  }

  .reset-icon:hover {
    transform: rotate(22.5deg);
  }

  .reset-icon :global(svg) {
    width: 16px;
    height: 16px;
  }

  .dialog-actions {
    display: flex;
    justify-content: flex-end;
    gap: 1rem;
    margin-top: 1rem;
  }

  @keyframes dialog-fade-in {
    from {
      opacity: 0;
    }
    to {
      opacity: 1;
    }
  }

  @keyframes dialog-fade-out {
    from {
      opacity: 1;
    }
    to {
      opacity: 0;
    }
  }

  @keyframes fade-in {
    from {
      opacity: 0;
      transform: translateY(1.618px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }
</style>
