<template>
<div class="c-notebook__entry c-ne has-local-controls"
     @dragover="changeCursor"
     @drop.capture="cancelEditMode"
     @drop.prevent="dropOnEntry"
>
    <div class="c-ne__time-and-content">
        <div class="c-ne__time">
            <span>{{ createdOnDate }}</span>
            <span>{{ createdOnTime }}</span>
        </div>
        <div class="c-ne__content">
            <div :id="entry.id"
                 class="c-ne__text"
                 tabindex="0"
                 :class="{ 'c-ne__input' : !readOnly }"
                 :contenteditable="!readOnly"
                 @blur="updateEntryValue($event)"
                 @keydown.enter.exact.prevent
                 @keyup.enter.exact.prevent="forceBlur($event)"
                 v-text="entry.text"
            >
            </div>
            <div class="c-snapshots c-ne__embeds">
                <NotebookEmbed v-for="embed in entry.embeds"
                               :key="embed.id"
                               :embed="embed"
                               :entry="entry"
                               @removeEmbed="removeEmbed"
                               @updateEmbed="updateEmbed"
                />
            </div>
        </div>
    </div>
    <div v-if="!readOnly"
         class="c-ne__local-controls--hidden"
    >
        <button class="c-icon-button c-icon-button--major icon-trash"
                title="Delete this entry"
                tabindex="-1"
                @click="deleteEntry"
        >
        </button>
    </div>
    <div v-if="readOnly"
         class="c-ne__section-and-page"
    >
        <a class="c-click-link"
           @click="navigateToSection()"
        >
            {{ result.section.name }}
        </a>
        <span class="icon-arrow-right"></span>
        <a class="c-click-link"
           @click="navigateToPage()"
        >
            {{ result.page.name }}
        </a>
    </div>
</div>
</template>

<script>
import NotebookEmbed from './NotebookEmbed.vue';
import { createNewEmbed } from '../utils/notebook-entries';
import Moment from 'moment';

export default {
    components: {
        NotebookEmbed
    },
    inject: ['openmct', 'snapshotContainer'],
    props: {
        domainObject: {
            type: Object,
            default() {
                return {};
            }
        },
        entry: {
            type: Object,
            default() {
                return {};
            }
        },
        result: {
            type: Object,
            default() {
                return {};
            }
        },
        selectedPage: {
            type: Object,
            default() {
                return {};
            }
        },
        selectedSection: {
            type: Object,
            default() {
                return {};
            }
        },
        readOnly: {
            type: Boolean,
            default() {
                return true;
            }
        }
    },
    computed: {
        createdOnDate() {
            return this.formatTime(this.entry.createdOn, 'YYYY-MM-DD');
        },
        createdOnTime() {
            return this.formatTime(this.entry.createdOn, 'HH:mm:ss');
        }
    },
    mounted() {
        this.dropOnEntry = this.dropOnEntry.bind(this);
    },
    methods: {
        addNewEmbed(objectPath) {
            const bounds = this.openmct.time.bounds();
            const snapshotMeta = {
                bounds,
                link: null,
                objectPath,
                openmct: this.openmct
            };
            const newEmbed = createNewEmbed(snapshotMeta);
            this.entry.embeds.push(newEmbed);
        },
        cancelEditMode(event) {
            const isEditing = this.openmct.editor.isEditing();
            if (isEditing) {
                this.openmct.editor.cancel();
            }
        },
        changeCursor() {
            event.preventDefault();
            event.dataTransfer.dropEffect = "copy";
        },
        deleteEntry() {
            this.$emit('deleteEntry', this.entry.id);
        },
        dropOnEntry($event) {
            event.stopImmediatePropagation();

            const snapshotId = $event.dataTransfer.getData('openmct/snapshot/id');
            if (snapshotId.length) {
                const snapshot = this.snapshotContainer.getSnapshot(snapshotId);
                this.snapshotContainer.removeSnapshot(snapshotId);
                this.entry.embeds.push(snapshot);
            } else {
                const data = $event.dataTransfer.getData('openmct/domain-object-path');
                const objectPath = JSON.parse(data);
                this.addNewEmbed(objectPath);
            }

            this.$emit('updateEntry', this.entry);
        },
        findPositionInArray(array, id) {
            let position = -1;
            array.some((item, index) => {
                const found = item.id === id;
                if (found) {
                    position = index;
                }

                return found;
            });

            return position;
        },
        forceBlur(event) {
            event.target.blur();
        },
        formatTime(unixTime, timeFormat) {
            return Moment.utc(unixTime).format(timeFormat);
        },
        navigateToPage() {
            this.$emit('changeSectionPage', {
                sectionId: this.result.section.id,
                pageId: this.result.page.id
            });
        },
        navigateToSection() {
            this.$emit('changeSectionPage', {
                sectionId: this.result.section.id,
                pageId: null
            });
        },
        removeEmbed(id) {
            const embedPosition = this.findPositionInArray(this.entry.embeds, id);
            this.entry.embeds.splice(embedPosition, 1);

            this.$emit('updateEntry', this.entry);
        },
        updateEmbed(newEmbed) {
            this.entry.embeds.some(e => {
                const found = (e.id === newEmbed.id);
                if (found) {
                    e = newEmbed;
                }

                return found;
            });

            this.$emit('updateEntry', this.entry);
        },
        updateEntryValue($event) {
            const value = $event.target.innerText;
            if (value !== this.entry.text && value.match(/\S/)) {
                this.entry.text = value;
                this.$emit('updateEntry', this.entry);
            }
        }
    }
};
</script>
