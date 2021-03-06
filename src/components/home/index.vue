<template>
    <v-container :fluid="$vuetify.breakpoint.mdAndDown" grid-list-md>
        <v-dialog v-model="ui.deleteDialog" max-width="400" lazy>
            <v-card>
                <v-card-title>
                    <i class="fas fa-trash fa-2x error--text"></i>
                    <div class="ml-3">
                        <h3 class="mb-0">Delete This Item?</h3>
                        <span class="text--secondary">You won't be able to access or recover it later</span>
                    </div>
                </v-card-title>
                <v-spacer/>
                <v-card-actions>
                    <v-spacer/>
                    <v-btn color="error" flat @click="onDeleteClick">
                        <span>Delete</span>
                    </v-btn>
                    <v-btn color="grey" flat @click="ui.deleteDialog = false">
                        <span>Cancel</span>
                    </v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>

        <v-layout row wrap align-center>
            <v-flex xs12 class="mb-3">
                <v-card flat class="transparent">
                    <v-card-title class="pa-0">
                        <div>
                            <h3 class="display-1">TagMania</h3>
                            <span class="text--secondary">Easily manage and organize your bookmarks, thoughts, files or anything you like</span>
                            <br>
                            <span class="text--secondary">
                                Made with
                                <i class="fas fa-heart red--text"></i> by
                                <a class="email-link" href="mailto:feras.da94@gmail.com">Feras Dawod</a>
                            </span>
                        </div>
                        <v-spacer/>
                        <div>
                            <v-btn @click="openSettings">
                                <v-icon small left>fas fa-cogs</v-icon>
                                <span>Settings</span>
                            </v-btn>
                        </div>
                    </v-card-title>
                </v-card>
            </v-flex>

            <v-flex md4 sm12>
                <v-text-field label="Search Term" v-model="search.name" box clearable hide-details/>
            </v-flex>

            <v-flex md4 sm12>
                <v-autocomplete
                    label="Type"
                    clearable
                    multiple
                    box
                    small-chips
                    deletable-chips
                    hide-details
                    :items="itemTypes"
                    item-text="text"
                    item-value="value"
                    v-model.lazy="search.types"
                />
            </v-flex>

            <v-flex md4 sm12>
                <v-autocomplete
                    label="Tags"
                    clearable
                    multiple
                    box
                    small-chips
                    deletable-chips
                    hide-details
                    :items="$store.state.tags"
                    item-text="name"
                    item-value="name"
                    v-model.lazy="search.tags"
                />
            </v-flex>

            <v-flex sm12>
                <v-card>
                    <v-card-title class="info white--text">
                        <h2 class="font-weight-regular">Latest Items</h2>
                        <v-spacer/>
                        <v-btn depressed color="white" class="ma-0" @click="navigateToAdd">
                            <v-icon small left>fas fa-plus</v-icon>
                            <span>New</span>
                        </v-btn>
                    </v-card-title>
                    <v-scroll-x-transition mode="out-in">
                        <v-card-text class="text-xs-center" v-if="loading" key="loading">
                            <v-progress-circular color="info" indeterminate/>
                        </v-card-text>
                        <v-list v-else-if="items.length" two-line key="data">
                            <v-menu v-model="ui.showMenu" :position-x="ui.x" :position-y="ui.y" absolute offset-y>
                                <v-list>
                                    <v-list-tile @click="onEditClick">
                                        <v-list-tile-avatar>
                                            <v-icon small>fas fa-pen</v-icon>
                                        </v-list-tile-avatar>
                                        <v-list-tile-title>Edit</v-list-tile-title>
                                    </v-list-tile>
                                    <v-list-tile class="error--text" @click="ui.deleteDialog = true">
                                        <v-list-tile-avatar>
                                            <v-icon small color="error">fas fa-trash</v-icon>
                                        </v-list-tile-avatar>
                                        <v-list-tile-title>Delete</v-list-tile-title>
                                    </v-list-tile>
                                </v-list>
                            </v-menu>

                            <template v-for="item in items">
                                <link-row :key="item._id" :item="item" @contextmenu="showContextMenu"/>
                            </template>
                        </v-list>

                        <v-card-text v-else class="text-xs-center" key="noData">
                            <i class="fas fa-frown fa-2x grey--text text-lighten-2"></i>
                            <br>
                            <div class="pt-2 headline text--secondary">No Data</div>
                        </v-card-text>
                    </v-scroll-x-transition>
                </v-card>
            </v-flex>
        </v-layout>
    </v-container>
</template>

<script>
import LinkRow from './partials/link-row';
import alert from '../../plugins/alert.service';

import _ from 'lodash';
import Fuse from 'fuse.js';

export default {
    components: {
        LinkRow,
    },
    data: () => ({
        ui: {
            showMenu: false,
            deleteDialog: false,
            x: null,
            y: null,

            activeItem: null,
        },
        items: [],
        loading: true,
        search: {
            name: null,
            types: [],
            tags: [],
        },
    }),

    computed: {
        itemTypes() {
            return [{ text: 'Links', value: 0 }, { text: 'Notes', value: 1 }, { text: 'Documents', value: 2 }];
        },
    },

    watch: {
        $route: function() {
            this.fetchLatest();
        },
        search: {
            handler: _.debounce(function() {
                this.fetchData();
            }, 500),
            deep: true,
        },
    },
    methods: {
        showContextMenu(payload) {
            payload.e.preventDefault();
            this.ui.showMenu = false;
            this.ui.x = payload.e.clientX;
            this.ui.y = payload.e.clientY;
            this.$nextTick(() => {
                this.ui.showMenu = true;
                this.ui.activeItem = payload.item;
            });
        },

        navigateToAdd() {
            this.$router.push({ name: 'add' });
        },

        onEditClick() {
            this.$router.push({
                name: 'add',
                query: { id: this.ui.activeItem._id },
            });
        },

        onDeleteClick() {
            this.$store.dispatch('deleteItem', this.ui.activeItem).then(() => {
                const index = this.items.indexOf(this.ui.activeItem);
                this.items.splice(index, 1);
                this.ui.deleteDialog = false;
                alert.success('Item Deleted');
            });
        },

        fetchData() {
            if (this.search.tags.length || this.search.types.length || this.search.name) this.fetchBySearch();
            else this.fetchLatest();
        },

        fetchLatest() {
            this.loading = true;
            this.$store
                .dispatch('getLatestItems')
                .then(response => {
                    this.items = response;
                    this.loading = false;
                })
                .catch(() => {
                    this.loading = false;
                });
        },

        fetchBySearch() {
            this.loading = true;
            this.$store
                .dispatch('getAllItems')
                .then(response => {
                    let items = response.filter(item => {
                        let result = true;

                        if (this.search.tags.length) result &= this.search.tags.every(tag => item.tags.includes(tag));
                        if (this.search.types.length) result &= this.search.types.includes(item.type);

                        return result;
                    });

                    if (this.search.name) {
                        var options = {
                            keys: ['name', 'link', 'content', 'tags'],
                            shouldSort: true,
                            tokenize: true,
                            findAllMatches: true,
                        };
                        const fuse = new Fuse(items, options);
                        items = fuse.search(this.search.name);
                    }

                    this.items = items;
                    this.loading = false;
                })
                .catch(() => {
                    this.loading = false;
                });
        },

        openSettings() {
            this.$router.push({ name: 'settings.index' });
        },
    },
    mounted() {
        this.fetchLatest();
    },
};
</script>

<style scoped>
.email-link {
    color: initial;
    text-decoration: none;
}
</style>
