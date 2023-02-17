<template>
    <div class="text-center min-h-screen">
        <h1 class="text-center my-2">Import History</h1>
        <form>
            <br />
            <div>
                <input ref="fileSelector" type="file" @change="fileChange" />
            </div>
            <div>
                <strong v-text="`Found ${itemsLength} items`" />
            </div>
            <div>
                <strong>Override: <input v-model="override" class="checkbox" type="checkbox" /></strong>
            </div>
            <br />
            <div>
                <progress :value="index" :max="itemsLength" />
                <div v-text="`Success: ${success} Error: ${error} Skipped: ${skipped}`" />
            </div>
            <br />
            <div>
                <a class="btn w-auto" @click="handleImport">Import</a>
            </div>
        </form>
    </div>
</template>
<script>
export default {
    data() {
        return {
            items: [],
            override: false,
            index: 0,
            success: 0,
            error: 0,
            skipped: 0,
        };
    },
    computed: {
        itemsLength() {
            return this.items.length;
        },
    },
    activated() {
        document.title = "Import History - Piped";
    },
    methods: {
        fileChange() {
            const file = this.$refs.fileSelector.files[0];
            file.text().then(text => {
                this.items = [];
                const json = JSON.parse(text);
                const items = json.watchHistory.map(video => {
                    return {
                        ...video,
                        watchedAt: video.watchedAt ?? 0,
                        currentTime: video.currentTime ?? 0,
                    };
                });
                this.items = items.sort((a, b) => b.watchedAt - a.watchedAt);
            });
        },
        handleImport() {
            if (window.db) {
                var tx = window.db.transaction("watch_history", "readwrite");
                var store = tx.objectStore("watch_history");
                this.items.forEach(item => {
                    const dbItem = store.get(item.videoId);
                    dbItem.onsuccess = () => {
                        if (dbItem.result && dbItem.result.videoId === item.videoId) {
                            if (!this.override) {
                                this.index++;
                                this.skipped++;
                                return;
                            }
                        }
                        try {
                            const request = store.put(JSON.parse(JSON.stringify(item)));
                            request.onsuccess = () => {
                                this.index++;
                                this.success++;
                            };
                            request.onerror = () => {
                                this.index++;
                                this.error++;
                            };
                        } catch (error) {
                            console.error(error);
                            this.index++;
                            this.error++;
                        }
                    };
                });
            }
        },
    },
};
</script>
