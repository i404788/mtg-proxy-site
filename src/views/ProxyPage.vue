<template>
    <div class="section">
        <HelpModal ref="helpModal" />

        <div class="columns">
            <div class="column col-3 col-sm-12" style="z-index:300;">
                <div id="config" class="form-group p-sticky">
                    <div class="form-group">
                        <textarea id="deck-input" class="form-input" v-model="config.decklist" autofocus
                            placeholder="4 Wild Nacatl&#10;4 Steppe Lynx&#10;0x Griselbrand&#10;4x Lightning Bolt&#10;3x Price of Progress&#10;4 Strip Mine (ATQ) 82d&#10;2 Tinker Bell - Giant Fairy&#10;4 Rafiki - Mysterious Sage&#10;&#10;// Sideboard&#10;Orim's Chant&#10;3x Rough // Tumble&#10;SB: dead/gone"></textarea>
                    </div>

                    <div class="form-group btn-group btn-group-block">
                        <button id="submit-decklist" class="btn btn-primary" @click="loadCardList()">{{ cards.length ? 'Update' : 'Submit' }}</button>
                        <button id="print" class="btn btn-block tooltip" @click="printList" :disabled="cards.length == 0" :data-tooltip="`${cardCountWhenPrinting.count} of ${cardCountWhenPrinting.bound} slots consumed.\nAssuming an 8.5x11 paper size.`"><span class="icon-print"></span> Print</button>
                    </div>

                    <div class="form-group btn-group btn-group-block">
                        <div id="slot-usage" class="bar">
                            <template v-for="index in 9" :key="index">
                                <div :class="`bar-item ${index <= cardCountWhenPrinting.overflow ? 'consumed' : 'unconsumed'}`" role="progressbar"></div>
                            </template>
                        </div>
                    </div>

                    <div class="spacer" style="height:0.4rem;"></div>
                    <div class="divider text-center" data-content="CONFIGURATION"></div>

                    <div class="columns">
                        <div class="column col-12">
                            <label class="form-switch">
                                <input type="checkbox" v-model="config.includeDigital">
                                <i class="form-icon"></i> Show Digital Printings
                            </label>
                        </div>

                        <div class="column col-12">
                            <label class="form-switch">
                                <input type="checkbox" v-model="config.includePromo">
                                <i class="form-icon"></i> Show Promo Printings
                            </label>
                        </div>

                        <div class="column col-12">
                            <label class="form-switch">
                                <input type="checkbox" v-model="config.includeBasics">
                                <i class="form-icon"></i> Include Basic Lands
                            </label>
                        </div>

                        <div class="column col-12 divider"></div>

                        <div class="column col-12">
                            <label class="form-label">
                                <i class="form-icon"></i> Print Scale
                                <select class="form-select select" v-model="config.scale" style="width:100%;">
                                    <option value="small">Wimpy (98%)</option>
                                    <option value="normal">Regular (100%)</option>
                                    <option value="large">Jacked (102%)</option>
                                </select>
                            </label>
                        </div>

                        <div class="column col-12">
                            <label class="form-switch">
                                <input type="checkbox" v-model="config.dfcBacks">
                                <i class="form-icon"></i> Print DFC Backs
                            </label>
                        </div>

                        <div class="column col-12 divider"></div>

                        <div class="column col-12">
                            <button class="btn p-centered" @click="$refs.helpModal.show()">Help?</button>
                        </div>

                        <div class="column col-12 divider"></div>
                    </div>
                </div>
            </div>

            <div class="column col-9 col-sm-12">
                <div class="empty" v-show="cards.length === 0 && errors.length === 0">
                    <div class="empty-icon">
                        <i class="icon icon-3x icon-search"></i>
                    </div>
                    <p class="empty-title h5" style="max-width: 25rem;">"I welcome and seek your ideas, but do not bring me small ideas; bring me big ideas to match our future."</p>
                    <p class="empty-subtitle">- Arnold Schwarzenegger</p>
                </div>

                <div id="input-errors" class="toast toast-error" v-show="errors.length > 0">
                    <button class="btn btn-clear float-right" @click="errors = []"></button>
                    <div>Some cards could not be identified.</div>
                    <ul>
                        <li v-for="(error, index) in errors" :key="index">{{ error }}</li>
                    </ul>
                </div>

                <div class="cards columns">
                    <div v-for="(card, index) in cards" :key="index" class="card-select column col-3 col-sm-6 mt-2" v-show="shouldShowCard(card)">
                        <div class="p-relative">
                            <ImageLoader class="card-image img-responsive" :src="card.selectedOption.url" placeholder="./card_back.jpg" :alt="card.name" />
                            <span class="card-quantity bg-primary text-light docs-shape s-rounded centered">{{ card.quantity }}x</span>
                            <select class="form-select select-sm mt-2" v-model="card.selectedOption" @change="updateSessionSet(card.name, card.selectedOption)">
                                <option v-for="(set, index) in card.setOptions" :value="set" :key="index" v-show="shouldShowSetOption(card, set)">{{ set.name }}</option>
                            </select>
                        </div>
                    </div>
                </div>

                <ArnoldsApproval id="arnold" :cards="cards" />
            </div>
        </div>
    </div>

    <div id="print-content" :class="'scale-' + config.scale">
        <template v-for="(card, index) in cards" :key="index">
            <img v-for="n in card.quantity" :key="n" :src="card.selectedOption.url" v-show="shouldShowCard(card)">
            <img v-for="n in card.quantity" :key="n" :src="card.selectedOption.urlBack" v-show="shouldShowCard(card, 'back')">
        </template>
    </div>
</template>

<script>
import { normalizeCardName } from '../helpers/CardNames.mjs';
import ImageLoader from '../components/ImageLoader.vue';
import HelpModal from '../components/HelpModal.vue';
import ArnoldsApproval from '../components/ArnoldsApproval';

// Chunk out the card list for quasi-lazy loading. Or at least loading that doesn't block the page rendering.
const ScryfallDatasetAsync = () => import('../../data/cards-minimized.json');

const basicLands = [
    'wastes',
    'forest', 'island', 'plains', 'swamp', 'mountain',
    'snow-covered forest', 'snow-covered island',
    'snow-covered plains', 'snow-covered swamp',
    'snow-covered mountain'];

export default {
    name: 'ProxyPage',
    components: {
        ImageLoader,
        HelpModal,
        ArnoldsApproval,
    },
    data() {
        return {
            config: {
                includeDigital: false,
                includePromo: false,
                includeBasics: false,
                dfcBacks: true,
                scale: 'normal',
                decklist: '',
            },
            sets: {},
            cards: [],
            errors: [],
            sessionSetSelections: {},
        }
    },
    computed: {
        cardCountWhenPrinting() {
            const count = this.cards.reduce((total, c) => {
                return total + (c.quantity * ((this.shouldShowCard(c, 'front') ? 1 : 0) + (this.shouldShowCard(c, 'back') ? 1 : 0)));
            }, 0);

            const overflow = count % 9;
            const bound = overflow == 0 ? count : count + (9 - count % 9);

            return {
                count,
                overflow,
                bound,
                percentage: Math.round(overflow / 9 * 100),
            };
        },
    },
    mounted() {
        // Trigger an immediate load of the card list + set names.
        this.loadSetList();
        this.loadConfig();
    },
    methods: {
        async loadSetList() {
            this.sets = (await ScryfallDatasetAsync()).sets;
        },
        shouldShowSetOption(card, option) {
            // FIXME: Need a better filter method to detect promo-only garbage.
            // This initial clause here is to tackle promo-only or if the user has a promo selected.
            if (card.setOptions.length <= 1 || card.selectedOption == option) {
                return true;
            }

            return (this.config.includeDigital || !option.isDigital) && (this.config.includePromo || !option.isPromo);
        },
        shouldShowCard(card, face = 'front') {
            if (card.isMeld) {
                return true
            }
        
            if (!this.config.includeBasics && card.isBasic) {
                return false;
            }

            if (face === 'back' && (card.selectedOption.urlBack === undefined || !this.config.dfcBacks)) {
                return false;
            }

            return true;
        },
        updateSessionSet(cardName, setOption) {
            this.sessionSetSelections[cardName] = setOption;
        },
        storeConfig() {
            localStorage.includeDigital = this.config.includeDigital;
            localStorage.includePromo = this.config.includePromo;
            localStorage.includeBasics = this.config.includeBasics;
            localStorage.dfcBacks = this.config.dfcBacks;
            localStorage.scale = this.config.scale;
        },
        loadConfig() {
            this.config.includeDigital = localStorage.includeDigital === 'true';
            this.config.includePromo = localStorage.includePromo === 'true';
            this.config.includeBasics = localStorage.includeBasics === 'true';
            this.config.dfcBacks = !(localStorage.dfcBacks === 'false');
            this.config.scale = localStorage.scale ?? 'normal';
        },
        printList() {
            this.storeConfig();
            window.print();
        },
        async loadCardList() {
            this.cards = [];
            this.errors = [];
            for (let line of this.config.decklist.split('\n')) {
                line = line.trim();

                // Different sites have different sideboard formats.
                // Look for the word "sideboard" or lines that start with a double slash and skip them.
                if (/Sideboard/i.test(line) || /^\/\//.test(line) || line === '') {
                    continue;
                }

                // Extract the quantity and card name.
                // Cockatrice prefixes lines with "SB:" for sideboard cards, so optionally matching that.
                // MTGA's export format puts the set and collector number in the line. ex. Arid Mesa (ZEN) 211
                let extract = /^(?:SB:\s)?(?:(\d+)?x?\s)?([^(]+)(?:\s\(.+\) .+)?$/i.exec(line);
                if (extract === null) {
                    this.errors.push(line);
                    console.warn(`Failed to parse line: ${line}`);
                    continue;
                }

                let [, quantity, inputCardName] = extract;

                if (quantity === undefined) {
                    quantity = 1;
                }

                // parseInt should be safe here since it's a digit extraction,
                // decimal numbers will just get roped into the cardName and fail.
                if (parseInt(quantity) <= 0) {
                    continue;
                }

                const cardName = normalizeCardName(inputCardName);

                const cardLookup = (await ScryfallDatasetAsync()).cards[cardName];

                if (!cardLookup) {
                    this.errors.push(line);
                    console.warn(`Failed to identify card on line: ${line}`);
                    continue;
                }

                const options = {
                    quantity: parseInt(quantity),
                    name: cardName,
                    inputName: inputCardName,
                    setOptions: cardLookup.map(option => {
                        let [ setCode, setNumber ] = option.s.split('|');
                        return {
                            name: `${this.sets[setCode]} (${setNumber})`,
                            url: option.f,
                            urlBack: option.b,
                            isDigital: option.d === 1,
                            isPromo: option.p === 1,
                            isMeld: option.m === 1,
                        };
                    }),
                    isBasic: basicLands.includes(cardName.toLowerCase()),
                    selectedUrl: '',
                    selectedOption: this.sessionSetSelections[cardName],
                };

                if (!options.selectedOption) {
                    // Set a default selection.
                    options.selectedOption = options.setOptions.filter(option => {
                        return !option.isDigital && !option.isPromo;
                    })?.[0] ?? options.setOptions[0];
                }

                this.cards.push(options);
            }
        },
    },
}
</script>

<style>
#deck-input {
    height: 20rem;
}

@media (max-width: 600px) {
    #deck-input {
        height: 10rem;
    }
}

#config {
    top: 0.6rem;
}

#slot-usage {
    border-collapse: collapse;
    height: 0.3rem;
}

#slot-usage .bar-item {
    border: 1px solid #bcc3ce;
    border-collapse: collapse;
    width: 11.1%;
}

#slot-usage .bar-item.consumed {
    background: #5755d9;
}

#slot-usage .bar-item.unconsumed {
    background: #eef0f3;
}

.card-quantity {
    font-size: 1.2rem;
    font-weight: 100;
    display: inline-block;
    position: absolute;
    bottom: 2.4rem;
    left: 0.6rem;
    padding: 0.2rem;
    line-height: 1rem;
}

.card-image {
    /* border-radius: 4.75% / 3.5%; */
    border-radius: 0.3rem;
}

#arnold {
    margin-top: 2.4em;
}

#input-errors ul li {
    margin-top: unset;
}

#print-content {
    display: none;
}

#print-content {
    line-height: 0;
}

#print-content img {
    width: 60mm;
    height: 85mm;
    margin: 0;
    padding: 0;
}

@media print {
    body {
        all: initial;
    }

    html, html * {
        all: unset;
        font-size: 0 !important;
        line-height: 0 !important;
    }

    .section, header {
        display: none !important;
    }

    body {
        margin: 0 !important;
    }

    @page {
        size: auto;
        margin-left: 1cm;
        margin-right: 1cm;
        margin-bottom: 5mm;
        margin-top: 10mm;
    }

    html {
        visibility: hidden;
    }

    #print-content {
        visibility: visible;
        display: block !important;
        line-height: 0;
    }

    #print-content img {
        width: 60mm;
        height: 85mm;
        margin: 0;
        padding: 0;
    }

    #print-content.scale-large img {
        width: calc(60mm * 1.02);
        height: calc(85mm * 1.02);
    }

    #print-content.scale-small img {
        width: calc(60mm * 0.98);
        height: calc(85mm * 0.98);
    }

    img {
        break-inside: avoid;
    }
}
</style>
