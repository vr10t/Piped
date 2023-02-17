<template>
    <div class="text-center">
        <form>
            <br />
            <div>
                <input ref="fileSelector" type="file" @change="fileChange" />
            </div>
            <br />
            <div>
                <strong v-if="items.length > 0" v-text="`Found ${itemsLength} items`" />
            </div>
            <div>
                <progress :value="index" :max="itemsLength" />
                <div v-text="`Success: ${success} Error: ${error} Skipped: ${skipped}`" />
            </div>
            <div>
                <a class="btn w-auto" @click="handleImport">Import</a>
            </div>
        </form>
        <br />
        <strong>Importing Subscriptions from YouTube</strong>
        <br />
        <div>
            Open
            <a href="https://takeout.google.com/takeout/custom/youtube">takeout.google.com/takeout/custom/youtube</a>
            <br />
            In "Select data to include", click on "All YouTube data included" and select only "subscriptions".
            <br />
            Create the export and download the zip file.
            <br />
            Extract subscriptions.csv from the zip file.
            <br />
            Select and import the file above.
        </div>
        <br />
        <strong>Importing Subscriptions from Invidious</strong>
        <br />
        <div>
            Open
            <a href="https://invidio.us/data_control">invidiou.us/data_control</a>
            <br />
            Click on any of the export options.
            <br />
            Select and import the file above.
        </div>
        <br />
        <strong>Importing Subscriptions from NewPipe</strong>
        <br />
        <div>
            Go to the Feed tab.
            <br />
            Click on the arrow on where it says "Subscriptions".
            <br />
            Save the file somewhere.
            <br />
            Select and import the file above.
        </div>
    </div>
</template>
<script>
export default {
    data() {
        return {
            items: [],
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
    methods: {
        fileChange() {
            const file = this.$refs.fileSelector.files[0];
            file.text().then(text => {
                this.items = [];
                // LibreTube
                if (text.includes("watchPositions")) {
                    const json = JSON.parse(text);
                    const items = json.watchHistory.map(video => {
                        return {
                            duration: video.duration,
                            thumbnail: video.thumbnailUrl,
                            title: video.title,
                            uploaderName: video.uploader,
                            uploaderUrl: video.uploaderUrl,
                            videoId: video.videoId,
                            watchedAt: parseInt(video.watchedAt),
                            currentTime: json.watchPositions.find(i => i.videoId === video.videoId)?.position ?? 0,
                        };
                    });
                    this.items = items.sort((a, b) => b.watchedAt - a.watchedAt);
                }
                // const json = JSON.parse(text);
                // const items = json.watchHistory.map(video => {
                //     return {
                //         duration: video.duration,
                //         thumbnail: video.thumbnailUrl,
                //         title: video.title,
                //         uploaderName: video.uploader,
                //         uploaderUrl: video.uploaderUrl,
                //         videoId: video.videoId,
                //         watchedAt: video.watchedAt,
                //         currentTime: video.currentTime,
                //     };
                // });
                // FreeTube
                if (text.startsWith(`{"videoId:`)) {
                    text = `[${text.replace(/\n/g, ", ").slice(0, -2)}]`;
                    let json = JSON.parse(text);
                    console.log(json);
                    this.items = json
                        .map(video => {
                            return {
                                duration: video.duration,
                                thumbnail: video.thumbnail,
                                title: video.title,
                                uploaderName: video.author,
                                uploaderUrl: video.authorUrl,
                                videoId: video.videoId,
                                watchedAt: video.watchedDate,
                                currentTime: video.currentTime,
                            };
                        })
                        .sort((a, b) => b.watchedAt - a.watchedAt);
                }
                // NewPipe
                // if (text.startsWith("SQLite Format 3")) {
                //     const db = new SQL.Database(text);
                //     const results = db.exec("SELECT * FROM subscriptions");
                //     this.items = results[0].values.map(video => {
                //         return {
                //             duration: video[3],
                //             thumbnail: video[4],
                //             title: video[5],
                //             uploaderName: video[6],
                //             uploaderUrl: video[7],
                //             videoId: video[8],
                //             watchedAt: video[9],
                //             currentTime: video[10],
                //         };
                //     });
                // }
                // Invidious
                if (text.startsWith(`{"subscriptions":`)) {
                    const json = JSON.parse(text);
                    console.log(json);
                    this.items = json.watch_history.map(video => {
                        return {
                            videoId: video,
                        };
                    });
                }
                console.log(this.items);
            });
        },
        handleImport() {
            if (window.db) {
                var tx = window.db.transaction("watch_history", "readwrite");
                var store = tx.objectStore("watch_history");
                this.items.forEach(item => {
                    console.log(item);
                    // Don't override invidious items since they don't have all the data
                    const dbItem = store.get(item.videoId);
                    dbItem.onsuccess = () => {
                        if (dbItem.result && dbItem.result.videoId === item.videoId) {
                            console.log("Skipping", item.videoId);
                            this.index++;
                            this.skipped++;
                            return;
                        }
                        try {
                            const request = store.put(JSON.parse(JSON.stringify(item)));
                            request.onsuccess = () => {
                                console.log("success");
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
console.log("import");
</script>
