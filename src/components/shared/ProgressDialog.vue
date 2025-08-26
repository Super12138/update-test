<script setup lang="ts">
import "mdui/components/linear-progress.js";
import { useI18n } from "vue-i18n";

defineProps<{ progress: number | undefined }>();
const open = defineModel<boolean>({ required: true });

const { t } = useI18n();
</script>

<template>
    <mdui-dialog
        headline="正在下载更新"
        description="下载完成以后应用会重启安装更新"
        :open="open"
        close-on-overlay-click="false"
        close-on-esc="false"
        ref="dialog"
    >
        <mdui-linear-progress :value="progress" min="0" max="1"></mdui-linear-progress>
    </mdui-dialog>
</template>
