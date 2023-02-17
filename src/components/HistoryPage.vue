<template>
    <h1 class="font-bold text-center" v-t="'titles.history'" />

    <div class="flex">
        <div>
            <button class="btn" v-t="'actions.clear_history'" @click="clearHistory" />

            <button class="btn mx-3" v-t="'actions.export_to_json'" @click="showModal = !showModal" />

            <button class="btn" v-t="'actions.import_from_json'" @click="$router.push('/history/import')" />
        </div>

        <div class="right-1">
            <SortingSelector by-key="watchedAt" @apply="order => videos.sort(order)" />
        </div>
    </div>

    <hr />

    <div class="video-grid">
        <VideoItem v-for="video in videos" :key="video.url" :item="video" />
    </div>

    <br />
    <ModalComponent v-if="showModal" @close="showModal = !showModal">
        <div class="min-w-max flex flex-col">
            <h2 class="text-xl font-bold mb-4">Export History</h2>
            <form>
                <div>
                    <label class="mr-2" for="export-format">Export as:</label>
                    <select class="select" id="export-format" v-model="exportAs">
                        <option
                            v-for="option in exportOptions"
                            :key="option"
                            :value="option"
                            v-text="formatField(option)"
                        />
                    </select>
                </div>
                <div v-if="exportAs === 'history'">
                    <label v-for="field in fields" :key="field" class="flex gap-2 items-center">
                        <input
                            class="checkbox"
                            type="checkbox"
                            :value="field"
                            v-model="selectedFields"
                            :disabled="field === 'videoId'"
                        />
                        <span v-text="formatField(field)" />
                    </label>
                </div>
            </form>
            <button class="btn mt-4" @click="handleExport">Export</button>
        </div>
    </ModalComponent>
</template>

<script>
import VideoItem from "./VideoItem.vue";
import SortingSelector from "./SortingSelector.vue";
import ModalComponent from "./ModalComponent.vue";

export default {
    components: {
        VideoItem,
        SortingSelector,
        ModalComponent,
    },
    data() {
        return {
            videos: [],
            exportVideos: [],
            showModal: false,
            exportOptions: ["playlist", "history"],
            exportAs: "playlist",
            fields: [
                "videoId",
                "title",
                "uploaderName",
                "uploaderUrl",
                "duration",
                "thumbnail",
                "watchedAt",
                "currentTime",
            ],
            selectedFields: [
                "videoId",
                "title",
                "uploaderName",
                "uploaderUrl",
                "duration",
                "thumbnail",
                "watchedAt",
                "currentTime",
            ],
        };
    },
    mounted() {
        (async () => {
            if (window.db && this.getPreferenceBoolean("watchHistory", false)) {
                var tx = window.db.transaction("watch_history", "readonly");
                var store = tx.objectStore("watch_history");
                const cursorRequest = store.index("watchedAt").openCursor(null, "prev");
                cursorRequest.onsuccess = e => {
                    const cursor = e.target.result;
                    if (cursor) {
                        const video = cursor.value;
                        this.videos.push({
                            url: "/watch?v=" + video.videoId,
                            title: video.title,
                            uploaderName: video.uploaderName,
                            uploaderUrl: video.uploaderUrl ?? "", // Router doesn't like undefined
                            duration: video.duration ?? 0, // Undefined duration shows "Live"
                            thumbnail: video.thumbnail,
                            watchedAt: video.watchedAt,
                        });
                        if (this.videos.length < 1000) cursor.continue();
                    }
                };
            }
        })();
    },
    activated() {
        document.title = "Watch History - Piped";
    },
    methods: {
        clearHistory() {
            if (window.db) {
                var tx = window.db.transaction("watch_history", "readwrite");
                var store = tx.objectStore("watch_history");
                store.clear();
            }
            this.videos = [];
        },
        async fetchAllVideos() {
            if (window.db) {
                var tx = window.db.transaction("watch_history", "readonly");
                var store = tx.objectStore("watch_history");
                const request = store.getAll();
                return new Promise((resolve, reject) => {
                    (request.onsuccess = e => {
                        const videos = e.target.result;
                        this.exportVideos = videos;
                        resolve();
                    }),
                        (request.onerror = e => {
                            reject(e);
                        });
                });
            }
        },
        handleExport() {
            if (this.exportAs === "playlist") {
                this.fetchAllVideos()
                    .then(() => {
                        this.exportAsPlaylist();
                    })
                    .catch(e => {
                        console.error(e);
                    });
            } else if (this.exportAs === "history") {
                this.fetchAllVideos()
                    .then(() => {
                        this.exportAsHistory();
                    })
                    .catch(e => {
                        console.error(e);
                    });
            }
        },
        exportAsPlaylist() {
            const dateStr = new Date().toISOString().split(".")[0];
            let json = {
                format: "Piped",
                version: 1,
                playlists: [
                    {
                        name: `Piped History ${dateStr}`,
                        type: "history",
                        visibility: "private",
                        videos: this.exportVideos.map(video => "https://youtube.com" + video.url),
                    },
                ],
            };
            this.download(JSON.stringify(json), `piped_history_${dateStr}.json`, "application/json");
        },
        exportAsHistory() {
            const dateStr = new Date().toISOString().split(".")[0];
            let json = {
                format: "Piped",
                version: 1,
                watchHistory: this.exportVideos.map(video => {
                    let obj = {};
                    this.selectedFields.forEach(field => {
                        obj[field] = video[field];
                    });
                    return obj;
                }),
            };
            this.download(JSON.stringify(json), `piped_history_${dateStr}.json`, "application/json");
        },
        formatField(field) {
            // camelCase to Title Case
            return field.replace(/([A-Z])/g, " $1").replace(/^./, str => str.toUpperCase());
        },
    },
};
</script>
