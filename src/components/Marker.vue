<template>
    <div id="editor-container">
        <div id="vditor"></div>
        <div v-if="loading" class="loading-overlay">
            <div class="loading-spinner"></div>
            <p>请稍等几分钟</p>
        </div>
        <div v-if="errorMessage" class="error-message">
            <p>{{ errorMessage }}</p>
            <button @click="clearError">关闭</button>
        </div>
    </div>
</template>

<style scoped>

.loading-overlay, .error-message {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000;
}

.error-message p {
    color: white;
    font-size: 20px;
    text-align: center;
}

.error-message button {
    margin-top: 20px;
    padding: 10px 20px;
    font-size: 16px;
    cursor: pointer;
}

.loading-spinner {
    width: 50px;
    height: 50px;
    border: 5px solid #f3f3f3;
    border-top: 5px solid #3498db;
    border-radius: 50%;
    animation: spin 2s linear infinite;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}
</style>

<script lang="ts">
import { defineComponent, ref, onMounted, Ref } from 'vue';
import axios, { AxiosResponse } from 'axios';
import Vditor from 'vditor';
import 'vditor/dist/index.css';

export default defineComponent({
    name: 'MarkdownEditor',
    setup() {
        const loading = ref(false);
        const vditor: Ref<Vditor | null> = ref(null);
        const errorMessage = ref('');

        function clearError() {
            errorMessage.value = '';
        }

        async function uploadAndConvertPDF() {
            const input = document.createElement('input');
            input.type = 'file';
            input.accept = 'application/pdf';
            input.onchange = async (event) => {
                const target = event.target as HTMLInputElement;
                const file = target.files ? target.files[0] : null;
                if (file) {
                    loading.value = true;
                    try {
                        const formData = new FormData();
                        formData.append('pdf_file', file);
                        formData.append('extract_images', 'true');
                        const response: AxiosResponse<any> = await axios.post('/api/convert', formData, {
                            headers: { 'Content-Type': 'multipart/form-data' }
                        });
                        handleConversionResponse(response.data);
                    } catch (error) {
                        errorMessage.value = '转换 PDF 时发生错误，请稍后再试。';
                        console.error('Error converting PDF to Markdown:', error);
                    } finally {
                        loading.value = false;
                    }
                }
            };
            input.click();
        }

        function handleConversionResponse(data: { markdown: string, images: { [key: string]: string } }) {
            if (vditor.value) {
                vditor.value.setValue(data.markdown);
                processImages(data.images);
            }
        }

        function processImages(images: { [key: string]: string }) {
            Object.keys(images).forEach(key => {
                const imageUrl = images[key];
                console.log(imageUrl);
            });
        }

        onMounted(() => {
            if (!vditor.value) {
                vditor.value = new Vditor('vditor', {
                    height: '100vh',
                    width: '100vw',
                    toolbar: getToolbarOptions(),
                    cache: { enable: true }
                });
            }
        });

        function getToolbarOptions() {
            return [
                {
                    name: 'pdf-to-md',
                    tip: 'PDF转markdown',
                    className: 'pdf-markdown-button',
                    icon: '<svg><use xlink:href="#vditor-icon-upload"></use></svg>',
                    click: uploadAndConvertPDF
                },
                '|',
                'headings', 'bold', 'italic', 'strike', 'link', 'list', 'ordered-list',
                'check', 'quote', 'line', 'code', 'inline-code', 'insert-before', 'insert-after',
                'table', 'undo', 'redo', 'code-theme', 'content-theme','edit-mode' , 'both' , 'preview' , 'outline'
            ];
        }
         // 添加CSS来调整提示气泡的位置
         const style = document.createElement('style');
            style.innerHTML = `
          .vditor-toolbar .vditor-tooltipped::after {
            top: 100%; /* 将提示气泡放到工具栏按钮的下方 */
            bottom: auto; /* 确保底部位置为自动 */
          }
        `;
            document.head.appendChild(style);

        return { loading, errorMessage, clearError };
    },
});
</script>
