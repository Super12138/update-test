<script setup lang="ts">
import { UPDATE_URL } from "@/interfaces/constants";
import type { GitHubRelease } from "@/interfaces/GitHubRelease";
import { useAutoUpdateStore } from "@/stores/settings/autoUpdate";
import { useFetch } from "@vueuse/core";
import MarkdownIt from "markdown-it";
import { snackbar } from "mdui";
import { lt } from "semver";
import { onUnmounted, ref, watch } from "vue";
import RichDialog from "./RichDialog.vue";
import { useI18n } from "vue-i18n";
import { check } from "@tauri-apps/plugin-updater";
import { relaunch } from "@tauri-apps/plugin-process";

const { t } = useI18n();
const markdownIt = new MarkdownIt({
    html: true,
    linkify: true,
    typographer: true,
});

const autoUpdateStore = useAutoUpdateStore(); // autoUpdateStore.enable是一个常量，用于控制是否启用自动更新

const dialogOpen = ref<boolean>(false);
const currentVersion = VERSION_NAME;
const newVersion = ref<string>("");

const { isFetching, error, data, abort, execute } = useFetch(UPDATE_URL, {
    immediate: false,
    timeout: 5000,
});

watch(
    () => autoUpdateStore.enable,
    (enabled) => {
        if (enabled) execute();
        else if (isFetching.value) abort();
    },
    { immediate: true },
);

watch(data, (d) => {
    if (!d || !autoUpdateStore.enable) return;
    const json = JSON.parse(d as string) as GitHubRelease;
    const remoteVersion = json.name;
    newVersion.value = json.name;
    if (lt(VERSION_NAME, remoteVersion)) {
        console.log("检测到新版本");
        dialogOpen.value = true;
    } else {
        console.log("当前版本已是最新");
        snackbar({ message: "当前已是最新版本" });
    }
});

watch(error, (err) => {
    if (!err || !autoUpdateStore.enable) return;
    if (err.name !== "AbortError") {
        // 过滤掉主动取消的错误 @DeepSeek
        console.error(err);
        snackbar({ message: `检查更新时出错（${err}）` });
    }
});

const openUpdateURLInBrowser = async () => {
    const update = await check();
    if (update) {
        alert("发现更新" + update.version);
        console.log(`found update ${update.version} from ${update.date} with notes ${update.body}`);
        let downloaded = 0;
        let contentLength = 0;
        // alternatively we could also call update.download() and update.install() separately
        await update.downloadAndInstall((event) => {
            switch (event.event) {
                case "Started":
                    alert("开始下载");
                    if (event.data.contentLength) {
                        contentLength = event.data.contentLength;
                    }
                    console.log(`started downloading ${event.data.contentLength} bytes`);
                    break;
                case "Progress":
                    downloaded += event.data.chunkLength;
                    snackbar({
                        message: `正在下载更新：${((downloaded / contentLength) * 100).toFixed(2)}%`,
                    });
                    console.log(`downloaded ${downloaded} from ${contentLength}`);
                    break;
                case "Finished":
                    alert("下载成功，开始安装");
                    console.log("download finished");
                    break;
            }
        });

                    alert("安装成功，即将重启");
        console.log("update installed");
        await relaunch();
    }
};

onUnmounted(() => {
    if (isFetching.value) abort();
});
</script>

<template>
    <RichDialog :headline="t('update-dialog.headline', { version: newVersion })"
        :description="t('update-dialog.description', { version: currentVersion })" v-model="dialogOpen"
        :close-on-overlay-click="false" @confirm="openUpdateURLInBrowser()">
    </RichDialog>
</template>
