<template>
    <div>
        <div v-if="showLoading && loading">
            <div class="panel panel-default">
                <div class="panel-body">
                    <i class="fa fa-btn fa-spinner fa-spin"></i>Loading
                </div>
            </div>
        </div>
        <div v-if="showResult && ! loading">
            <!-- Search Field Panel -->
            <div v-if="showSearch" class="panel panel-default panel-flush" style="border: 0;">
                <div class="panel-body">
                    <!-- Search Field -->
                    <div class="m-b-none">
                        <div class="col-md-12 p-none">
                            <input type="text" class="form-control" v-model="searchQuery"
                                   :placeholder="searchPlaceholderText" @keyup.enter="newSearch()">
                        </div>
                    </div>
                </div>
            </div>

            <div class="panel" :class="[emptyResult && showSearchResults ? 'panel-warning' : 'panel-default']">
                <div class="panel-heading">
                    {{ name }} <span v-if="showTotalResults">({{totalResults}})</span>

                    <div class="pull-right">
                        <slot name="header-buttons">
                        </slot>
                    </div>
                    <div class="clearfix"></div>
                </div>

                <!--Search Spinner -->

                <div class="panel-body" v-if="searching">
                    <i class="fa fa-btn fa-spinner fa-spin"></i>
                    Searching
                </div>

                <div class="panel-body" v-else>
                    <!-- Empty search results -->
                    <div v-if="emptyResult && showSearchResults">
                        {{ emptySearchResultsText }}
                    </div>

                    <!-- Empty results -->
                    <div v-if="emptyResult && ! showSearchResults">
                        {{ emptyResultsText }}
                    </div>

                    <!-- Table content -->
                    <div v-if="results.length > 0">
                        <slot name="table-content" :results="results" :text="name">
                        </slot>
                    </div>

                    <!-- Pagination -->
                    <div>
                        <ul v-if="paginator && totalPages > 1" class="pagination pagination-sm">
                            <template v-for="page in pages">
                                <li :class="[page === currentPage ? 'active' : '', isDisabledPage(page) ? 'disabled' : '']">
                                    <a href="#" @click.prevent="view(page)">
                                        {{ page }}
                                    </a>
                                </li>
                            </template>
                        </ul>
                    </div>
                </div>
            </div>
        </div>

    </div>
</template>

<style>
</style>

<script>
    export default{
        props: {
            // Name of the table.
            name: {
                type: String,
                default: 'Table'
            },
            // Boolean check to show the search field. Defaulted to show search field.
            showSearch: {
                type: Boolean,
                default: true
            },
            // Boolean check to show the loading panel. Defaulted to show loading panel.
            showLoading: {
                type: Boolean,
                default: true
            },
            // Boolean check to show the results panel when empty. Defaulted to show panel even if the results is empty.
            showIfEmpty: {
                type: Boolean,
                default: true
            },
            showTotalResults: {
                type: Boolean,
                default: true
            },
            // Text for the placeholder of search field.
            searchPlaceholderText: {
                type: String,
                default: 'Search...'
            },
            // Text to display when resource is empty.
            emptyResultsText: {
                type: String,
                default: 'No results found.'
            },
            // Text to display when search returns empty results.
            emptySearchResultsText: {
                type: String,
                default: 'No results matched the given criteria.'
            },
            // The url to the resource. May or may not be paginated
            resourceUrl: {
                type: String,
                required: true
            },
            // (Optional) Name of event to reset the contents of the paginated table. If not specified, paginated table
            // will not listen to this event.
            resetTableEvent: {
                type: String,
                default: null
            },
            // (Optional) Name of event where paginated table emits the total number of results. If not specified,
            // paginated table will not emit this event.
            updatedTableEvent: {
                type: String,
                default: null
            },
            // (Optional) Name of event where paginated table emits the list of displayed results. If not specified,
            // paginated table will not emit this event.
            updatedResultsEvent: {
                type: String,
                default: null
            }
        },

        data() {
            return {
                disabledPage: '...',
                searchQuery: '',
                searching: false,
                showSearchResults: false,
                loading: true,
                results: [],
                paginator: null,
                isReset: false,
            }
        },

        computed: {

            showResult: function () {
                if (this.results.length === 0 && !this.showIfEmpty) {
                    return false;
                }
                return true;
            },

            emptyResult: function () {
                return this.results.length === 0;
            },

            totalResults: function () {
                return this.paginator ? this.paginator.total : this.results.length;
            },

            currentPage: function () {
                return this.paginator ? this.paginator.current_page : 0;
            },

            totalPages: function () {
                return this.paginator ? this.paginator.total_pages : 0;
            },

            pages: function () {
                if (this.totalPages <= 10) {
                    return this.totalPages;
                }

                var pageIndices = [];
                var midPage = this.currentPage;
                var startPageRange = 1;
                var endPageRange = this.totalPages;
                pageIndices.push(1);
                if (midPage - 1 <= 5) {
                    startPageRange = 2;
                    midPage = 6;
                } else {
                    startPageRange = this.currentPage - 3;
                }
                var endPageDifference = this.totalPages - midPage;
                if (endPageDifference <= 4) {
                    startPageRange = startPageRange - (4 - endPageDifference);
                } else {
                    endPageRange = midPage + 3;
                }
                if (startPageRange !== 2) {
                    pageIndices.push(this.disabledPage);
                }
                pageIndices = pageIndices.concat(_.range(startPageRange, endPageRange));
                if (endPageRange !== this.totalPages) {
                    pageIndices.push(this.disabledPage);
                }
                pageIndices.push(this.totalPages);
                return pageIndices;
            }
        },

        created() {
            this.view();

            /**
             * Reset paginated table contents.
             */
            if (this.resetTableEvent) {
                this.eventHub.$on(this.resetTableEvent, () => {
                    this.isReset = true;
                    this.getResults();
                });
            }
        },

        beforeDestroy() {
            this.eventHub.$off(this.resetTableEvent);
        },

        mounted() {
            // Disable clicks from disabled pages.
            $('.pagination .disabled a').on('click', function (e) {
                e.preventDefault();
            });
        },

        methods: {

            /**
             * @return {boolean} True if page is disabled, e.g. '...'.
             */
            isDisabledPage(page) {
                return page === this.disabledPage;
            },

            /**
             * Process new search request.
             */
            newSearch() {
                // Set state to search, to display searching spinner.
                this.searching = true;
                this.showSearchResults = true;
                this.view();
            },

            /**
             * View the selected paginated results.
             */
            view(page) {
                if (this.isDisabledPage(page)) {
                    return;
                }

                page = page ? page : 1;
                // Send search query only when in search state.
                if (this.showSearchResults && this.searchQuery) {
                    this.searchResults(page, this.searchQuery);
                } else {
                    this.getResults(page);
                }
            },

            /**
             * Get the results from url.
             */
            getResults(page) {
                var uri = new URI(this.resourceUrl);
                uri.addSearch('page', page || 1);

                axios.get(uri.href())
                        .then(response => {
                            this.updateResults(response.data);
                        })
                        .catch((error) => {
                            var response = error.response;
                            this.error(response.data.error.message);
                        })
                        .finally(() => {
                            this.emitTableUpdatedEvent();
                            this.searching = false;
                            this.loading = false;
                            if (this.isReset) {
                                this.isReset = false;
                                this.searchQuery = '';
                                this.showSearchResults = false;
                            }
                        });
            },

            /**
             * Search for results on query.
             */
            searchResults(page, query) {
                var uri = new URI(this.resourceUrl);
                uri.addSearch('page', page || 1);
                uri.addSearch('query', query);

                axios.get(uri.href())
                        .then(response => {
                            this.updateResults(response.data);
                        })
                        .catch((error) => {
                            var response = error.response;
                            this.error(response.data.error.message);
                        })
                        .finally(() => {
                            this.searching = false;
                        });
            },

            /**
             * Update the list of results to be displayed on the paginated table.
             * @param responseData
             */
            updateResults(responseData) {
                var resultData = responseData.data;
                if (resultData) this.results = resultData;
                this.paginator = responseData.paginator;
                if (this.updatedResultsEvent) {
                    this.eventHub.$emit(this.updatedResultsEvent, this.results);
                }
            },

            /**
             * Emit an event that the table contents are updated.
             */
            emitTableUpdatedEvent() {
                if (this.updatedTableEvent) {
                    var totalResults = this.paginator ? this.paginator.total : this.results.length;
                    this.eventHub.$emit(this.updatedTableEvent, totalResults);
                }
            }
        }
    }
</script>

<style>
    .disabled a {
        cursor: default;
        pointer-events: none;
    }

    .pagination > li > a:focus,
    .pagination > li > span:focus {
        background-color: #fff;
    }
</style>