<template>
    <div id="app-inner">
        <h3>Interior Design / Adjacency Matrix Diagram Editor</h3>

        <div id="editor-wrap">
            <div id="editor-outer" class="row">
                <div id="editor" class="haft">
                    <textarea cols="30" rows="8" v-model="rawRooms"></textarea>
                    <p class="description">
                        - Input your room list, one room one line.<br />
                        - Your work will be auto saved in your local browser or
                        you can download state file and load later.<br />
                        - You want to read
                        <a
                            href="https://carolynjeanmatthews.com/2017/02/07/adjacency-matrix-decoded/"
                            >ADJACENCY MATRIX DECODED</a
                        >
                        by Carolyn Jean Matthews<br />
                        -
                        <a
                            href="https://github.com/olragon/adjacency-matrix-diagram"
                            >Open source</a
                        >
                    </p>

                    <div>
                        <label>
                            <input type="color" v-model="inactiveTextColor" />
                            Inactive text color
                        </label>

                        <label>
                            <input type="color" v-model="activeTextColor" />
                            Active text color
                        </label>
                    </div>

                    <div>
                        <button @click.prevent="loadSamples">
                            Need a sample?
                        </button>
                    </div>

                    <div>
                        <button
                            v-if="rooms.length > 0"
                            @click.prevent="downloadSvg"
                        >
                            Download SVG image
                        </button>
                        <button
                            v-if="rooms.length > 0"
                            @click.prevent="downloadState"
                        >
                            Download current state
                        </button>
                    </div>

                    <div>
                        <label
                            >Load saved state
                            <input
                                type="file"
                                id="file"
                                @change="handleStateFile"
                        /></label>
                    </div>
                </div>

                <div id="diagram" class="haft">
                    <svg
                        :width="svgSize.width"
                        :height="svgSize.height"
                        id="svg-image"
                    >
                        <g
                            v-for="(r, ri) in modelRooms"
                            :key="'room-' + ri"
                            :transform="r.g.transform"
                            class="room-wrap"
                            :class="{ active: isActive({ r, ri }) }"
                        >
                            <line
                                :x1="r.l1.x1"
                                :y1="r.l1.y1"
                                :x2="r.l1.x2"
                                :y2="r.l1.y2"
                                class="line"
                            />
                            <text
                                :y="height / 2 + 5"
                                :style="{
                                    fill: isActive({ r, ri })
                                        ? activeTextColor
                                        : inactiveTextColor,
                                }"
                            >
                                <tspan>{{ r.name }}</tspan>
                            </text>
                            <line
                                v-if="r.l2"
                                :x1="r.l2.x1"
                                :y1="r.l2.y1"
                                :x2="r.l2.x2"
                                :y2="r.l2.y2"
                                class="line"
                            />
                        </g>

                        <g
                            v-for="(r, ri) in modelRects"
                            :key="`cell-${ri}`"
                            :transform="r.g.transform"
                            class="rect-wrap"
                            @mouseover="handleMouseOverRect({ r, ri })"
                            @mouseout="handleMouseOutRect({ r, ri })"
                            style="pointer-events: bounding-box"
                            @click="changeState({ r, ri })"
                        >
                            <rect
                                :x="r.rect.x"
                                :y="r.rect.y"
                                :width="r.rect.width"
                                :height="r.rect.height"
                                class="rect"
                                :style="{ fill: r.code && r.code.color }"
                            ></rect>
                            <text
                                v-if="r.code"
                                :x="r.rect.width / 2"
                                :y="r.rect.height / 2"
                                :transform="`rotate(-45 ${r.rect.width / 2} ${
                                    r.rect.height / 2
                                })`"
                                dy="0.35em"
                                text-anchor="middle"
                                font-size="11"
                                class="cell-code"
                            >
                                {{ r.code.code }}
                            </text>
                        </g>

                        <g :transform="`translate(${legendX}, 0)`">
                            <g
                                v-for="(c, ci) in codes"
                                :key="`legend-${ci}`"
                                :transform="`translate(0,${ci * legendRowHeight})`"
                            >
                                <rect
                                    :x="0"
                                    :y="0"
                                    :width="legendCodeWidth"
                                    :height="legendRowHeight - 4"
                                    class="swatch"
                                    :style="{ fill: c.color }"
                                ></rect>
                                <text
                                    :x="legendCodeWidth / 2"
                                    :y="(legendRowHeight - 4) / 2"
                                    dy="0.35em"
                                    text-anchor="middle"
                                >
                                    <tspan>{{ c.code || "(blank)" }}</tspan>
                                </text>
                                <text
                                    :x="legendCodeWidth + baseLengthUnit"
                                    :y="(legendRowHeight - 4) / 2"
                                    dy="0.35em"
                                >
                                    <tspan>{{ c.label }}</tspan>
                                </text>
                            </g>
                        </g>
                    </svg>
                </div>
            </div>
        </div>
    </div>
</template>

<style>
.room-wrap text {
    fill: #cacaca;
}

.line {
    stroke: #000;
    stroke-width: 1px;
}

.triangle {
    stroke: black;
    fill: white;
}

.rect {
    fill: #fff;
    stroke: black;
    cursor: pointer;
}

.cell-code {
    pointer-events: none;
    user-select: none;
}

.swatch {
    stroke: #000;
}

.active text {
    fill: #000;
}

.row {
    display: flex;
}

.haft {
    width: 50%;
    padding: 5px;
}

#editor textarea {
    width: 100%;
}

#diagram {
    overflow-x: auto;
}
</style>

<script>
export default {
    data() {
        return {
            codes: [
                {
                    code: "1D",
                    label: "Must connect through a door",
                    color: "#CEEED0",
                },
                { code: "1W", label: "Must share a wall", color: "#E5F0DB" },
                {
                    code: "2",
                    label: "Must be on the same level",
                    color: "#E0EAF6",
                },
                {
                    code: "X",
                    label: "Must be kept apart (no shared wall, no door)",
                    color: "#F6C9CE",
                },
                { code: "", label: "No requirement", color: "#FFFFFF" },
            ],
            height: 30,
            baseLengthUnit: 10,
            legendRowHeight: 26,
            legendCodeWidth: 64,
            rooms: [],
            states: {},
            currentRooms: [],
            rawRooms: "",
            inactiveTextColor: "#cacaca",
            activeTextColor: "#000000",
        };
    },
    computed: {
        allStates() {
            return this.codes.map((c) => c.code);
        },
        length() {
            if (this.rooms.length <= 0) return 0;
            let rooms = this.rooms
                .slice()
                .sort((a, b) => b.name.length - a.name.length);
            return rooms[0].name.length;
        },
        modelRooms() {
            return this.rooms.map((r, ri) => {
                let isLast = ri === this.rooms.length - 1;
                r.g = {
                    transform: `translate(0,${this.height * ri})`,
                };
                r.l1 = {
                    x1: 0,
                    y1: 0,
                    x2: this.length * this.baseLengthUnit,
                    y2: 0,
                };
                if (isLast) {
                    r.l2 = {
                        x1: 0,
                        y1: this.height,
                        x2: this.length * this.baseLengthUnit,
                        y2: this.height,
                    };
                }
                return r;
            });
        },
        modelRects() {
            let numLayers = this.rooms.length - 1;
            let rectHeight = this.height / Math.SQRT2;
            let rects = [];
            for (let _i = 0; _i < numLayers; _i++) {
                for (let _j = 0; _j < numLayers - _i; _j++) {
                    let gx =
                        this.length * this.baseLengthUnit +
                        this.height / 2 +
                        (_j * this.height) / 2;
                    let gy =
                        _i * this.height +
                        this.height / 2 +
                        (_j * this.height) / 2;
                    let rect = {
                        from: this.rooms[_i],
                        to: this.rooms[_j + _i + 1],
                        g: {
                            transform: `translate(${gx},${gy}) rotate(45 0 0)`,
                        },
                        rect: {
                            x: 0,
                            y: 0,
                            width: rectHeight,
                            height: rectHeight,
                        },
                        gx,
                        gy,
                    };
                    let stateKey = `${rect.from.name}-${rect.to.name}`;
                    if (this.states[stateKey]) {
                        rect.state = this.states[stateKey];
                        rect.code = this.codes.find(
                            (c) => c.code === rect.state
                        );
                    }
                    rects.push(rect);
                }
            }
            return rects;
        },
        canStore() {
            return storageAvailable("localStorage");
        },
        appState() {
            return {
                states: this.states,
                rawRooms: this.rawRooms,
                inactiveTextColor: this.inactiveTextColor,
                activeTextColor: this.activeTextColor,
            };
        },
        legendX() {
            // to the right of the matrix's rightmost tip, so it never overlaps
            return (
                this.length * this.baseLengthUnit +
                (this.rooms.length * this.height) / 2 +
                2 * this.baseLengthUnit
            );
        },
        svgSize() {
            const rects = this.modelRects || [];
            if (rects.length === 0) {
                return { width: 0, height: 0 };
            }
            const longestLabel = Math.max(
                ...this.codes.map((c) => c.label.length)
            );
            const width =
                this.legendX +
                this.legendCodeWidth +
                (longestLabel + 2) * this.baseLengthUnit;
            const height = Math.max(
                Math.max(...rects.map((r) => r.gy || 0)) + 50,
                this.codes.length * this.legendRowHeight
            );
            return { width, height };
        },
    },
    created() {
        this.loadState();
    },
    watch: {
        rawRooms(val) {
            if (val) {
                this.rooms = val
                    .trim()
                    .split("\n")
                    .filter((v, i, a) => a.indexOf(v) === i)
                    .map((txt) => {
                        return {
                            name: txt.trim(),
                        };
                    });
                this.saveState();
            } else {
                this.rooms = [];
            }
        },
    },
    methods: {
        loadSamples() {
            this.rawRooms = `Entry Foyer\nDinning Room\nMaster Bedroom\nLiving Room\nMaster Bathroom`;
            this.states = {};
        },
        changeState({ r }) {
            let key = `${r.from.name}-${r.to.name}`;
            let state = this.allStates[0];
            console.log(`changeState: ${key} ${state}`);
            if (this.states[key]) {
                let curStateIndex = this.allStates.indexOf(this.states[key]);
                let nextStateIndex =
                    curStateIndex < this.allStates.length - 1
                        ? curStateIndex + 1
                        : 0;
                state = this.allStates[nextStateIndex];
            }
            this.states[key] = state;
            this.saveState();
        },
        handleMouseOverRect({ r }) {
            this.currentRooms = [r.from.name, r.to.name];
        },
        isActive({ r }) {
            return this.currentRooms && this.currentRooms.includes(r.name);
        },
        handleMouseOutRect() {
            this.currentRooms = [];
        },
        saveState() {
            if (this.canStore) {
                window.localStorage.setItem(
                    "appState",
                    JSON.stringify(this.appState)
                );
            }
        },
        loadState(rawState) {
            if (!rawState && this.canStore) {
                rawState = window.localStorage.getItem("appState");
            }
            if (rawState) {
                try {
                    let savedState = JSON.parse(rawState);
                    this.rawRooms = savedState.rawRooms;
                    Object.keys(savedState.states).forEach((k) => {
                        this.states[k] = savedState.states[k];
                    });
                } catch (err) {
                    alert("Cannot load saved state");
                }
            }
        },
        createDownloadLink(data, fileName) {
            let linkEl = document.createElement("a");
            linkEl.setAttribute("href", data);
            linkEl.setAttribute("download", fileName);
            return linkEl;
        },
        downloadState() {
            let stateTxt =
                "data:text/json;charset=utf-8," +
                encodeURIComponent(JSON.stringify(this.appState));
            let linkEl = this.createDownloadLink(
                stateTxt,
                "adjacency-matrix-diagram.json"
            );
            linkEl.click();
            linkEl.remove();
        },
        handleStateFile(evt) {
            let input = evt.target;
            if (input.files.length === 0) {
                return alert(
                    "Error! You must select adjacency-matrix-diagram JSON state file."
                );
            }
            if (!input.files[0].type.match(/json/)) {
                return alert("Error! It's not JSON file.");
            }
            let vm = this;
            let reader = new FileReader();
            reader.onload = function () {
                let text = reader.result;
                vm.loadState(text);
                alert("Success! State loaded.");
                input.value = "";
            };
            reader.readAsText(input.files[0]);
        },
        downloadSvg() {
            let svg = document.getElementById("svg-image");
            let serializer = new XMLSerializer();
            let source = serializer.serializeToString(svg);
            if (
                !source.match(
                    /^<svg[^>]+xmlns="http:\/\/www\.w3\.org\/2000\/svg"/
                )
            ) {
                source = source.replace(
                    /^<svg/,
                    '<svg xmlns="http://www.w3.org/2000/svg"'
                );
            }
            if (
                !source.match(/^<svg[^>]+"http:\/\/www\.w3\.org\/1999\/xlink"/)
            ) {
                source = source.replace(
                    /^<svg/,
                    '<svg xmlns:xlink="http://www.w3.org/1999/xlink"'
                );
            }
            // add css
            source = source.replace(
                /id="svg-image">/,
                'id="svg-image"><style>.room-wrap text{fill:#cacaca}.line{stroke:#000;stroke-width:1px}.triangle{stroke:#000;fill:#fff}.rect{fill:#fff;stroke:#000;cursor:pointer}.cell-code{pointer-events:none;user-select:none}.swatch{stroke:#000}.active text{fill:#000}.row{display:-webkit-box;display:-ms-flexbox;display:flex}.haft{width:50%;padding:5px}#editor textarea{width:100%}#diagram{overflow-x:auto}</style>'
            );
            source = '<?xml version="1.0" standalone="no"?>\r\n' + source;
            source =
                "data:text/xml;charset=utf-8," + encodeURIComponent(source);
            let linkEl = this.createDownloadLink(
                source,
                "adjacency-matrix-diagram.svg"
            );
            linkEl.click();
            linkEl.remove();
        },
    },
};

function storageAvailable(type) {
    try {
        var storage = window[type],
            x = "__storage_test__";
        storage.setItem(x, x);
        storage.removeItem(x);
        return true;
    } catch (e) {
        return (
            e instanceof DOMException &&
            // everything except Firefox
            (e.code === 22 ||
                // Firefox
                e.code === 1014 ||
                // test name field too, because code might not be present
                // everything except Firefox
                e.name === "QuotaExceededError" ||
                // Firefox
                e.name === "NS_ERROR_DOM_QUOTA_REACHED") &&
            // acknowledge QuotaExceededError only if there's something already stored
            storage.length !== 0
        );
    }
}
</script>
