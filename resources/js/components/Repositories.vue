<template>
    <div class="row">
        <div class="col-lg-4 col-md-5">
            <aside class="px-3 px-2 border rounded bg-white">
                <header class="pt-3">
                    <h4 class="Filter mb-0">Filter</h4>
                </header>
                <hr>
                <section class="name-section">
                    <div class="form-group">
                        <label for="filter">Find by:</label>
                        <select id="filter" class="form-control form-control-sm" v-model="sort">
                            <option value="">Select option:</option>
                            <option value="name">By name:</option>
                            <option value="owner">By owner:</option>
                        </select>
                        <input type="text" id="owner" v-model="search" class="form-control form-control-sm my-2"
                               placeholder="Insert word" :disabled="sort.trim().length < 1">
                    </div>
                </section>
                <section class="star-section">
                    <div class="form-group">
                        <label for="star">By stars:</label>
                        <input type="number" step="1" min="0" id="star" v-model="star"
                               class="form-control form-control-sm">
                    </div>
                </section>
                <section class="reset-button text-center my-2">
                    <button type="button" @click="resetFilter" class="btn btn-outline-primary">
                        Reset
                    </button>
                </section>
            </aside>
        </div>
        <div class="col-lg-8 col-md-7 mt-md-0 mt-2">
            <div class="px-3 px-2 border rounded bg-white">
                <header class="pt-3 text-center">
                    <h4 class="mb-0">The most trending Github repositories for last week</h4>
                </header>
                <hr>
                <section class="orderBy-section">
                    <div class="form-group">
                        <select id="orderBy" class="form-control form-control-sm" v-model="orderBy">
                            <option :value="{}">Order by default</option>
                            <option v-for="order in orders" :value="order.value">Order by {{ order.title }}</option>
                        </select>
                    </div>
                </section>
                <section v-if="filteredRepositories.length > 0 && !loading" class="results-section">
                    <div class="w-100">
                        <div v-for="(repository, index) in filteredRepositories" v-bind:key="index"
                             class="py-2 border-bottom">
                            <div class="card">
                                <div class="card-header">
                                    <h5 class="mb-0 text-center"><a :href="repository.html_url" target="_blank"
                                                                    class="text-dark text-decoration-none">{{
                                        repository.name.toUpperCase() }}</a></h5>
                                </div>
                                <div class="card-body">
                                    <div class="separate-details">
                                        <div class="text-center">
                                            <img :src="repository.owner.avatar_url" :alt="repository.owner.login"
                                                 width="80" class="rounded-circle">
                                        </div>
                                        <div class="text-left mt-2">
                                            <p class="mb-1"><b>ID:</b> {{ repository.owner.id}}</p>
                                            <p class="mb-1"><b>Login:</b> {{ repository.owner.login}}</p>
                                            <p class="mb-1"><b>Owner github URI:</b> {{ repository.owner.html_url}}</p>
                                            <p class="mb-1"><b>Description:</b> {{ repository.description }}</p>
                                            <p class="mb-1"><b>Stars:</b> {{ repository.watchers }}</p>
                                        </div>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                </section>
                <section v-else-if="filteredRepositories.length < 1 && !loading" class="results-section">
                    <h5 class="my-5 text-center">No repositories matching your desire</h5>
                </section>
                <section v-else class="loading-section">
                    <h3 class="my-4 text-center">Loading...</h3>
                </section>
            </div>
        </div>
    </div>
</template>

<script>

    export default {

        data() {
            return {
                sort: '',
                search: '',
                star: 0,
                date: '',
                repositories: [],
                loading: true,
                orderBy: {},
                orders: [
                    {
                        title: 'name (desc)',
                        value: {
                            name: 'desc'
                        }
                    },
                    {
                        title: 'name (asc)',
                        value: {
                            name: 'asc'
                        }
                    },
                    {
                        title: 'owner ID (desc)',
                        value: {
                            owner: 'desc'
                        }
                    },
                    {
                        title: 'owner ID (asc)',
                        value: {
                            owner: 'asc'
                        }
                    },
                    {
                        title: 'stars (desc)',
                        value: {
                            watchers: 'desc'
                        }
                    },
                    {
                        title: 'stars (asc)',
                        value: {
                            watchers: 'asc'
                        }
                    }
                ]
            }
        },

        mounted() {
            this.fetchRepositories();
        },

        watch: {

            orderBy() {

                // This watch property is created only to send request to Github API for loading sorted repositories.
                // Sorting will still work correctly if someone removes this property.

                if (this.orderBy !== {} && Object.keys(this.orderBy)[0] === 'watchers') {

                    const params = {
                        sort: 'stars',
                        order: this.orderBy[Object.keys(this.orderBy)[0]]
                    };

                    return this.fetchRepositories(params);

                }

            }

        },

        computed: {

            filteredRepositories() {

                const sort = this.sort;

                let results = (sort.trim().length > 0) ? this.repositories.filter(repository => {
                    return sort === 'owner' ? repository[sort]['login'].toLowerCase().includes(this.search.toLowerCase()) : repository[sort].toLowerCase().includes(this.search.toLowerCase());
                }) : this.repositories;

                results = results.filter(result => result['watchers'] >= this.star);

                if (this.orderBy !== {}) {

                    const key = Object.keys(this.orderBy)[0];
                    const value = this.orderBy[key];

                    return (key === 'owner') ? _.orderBy(results, ['owner.id'], [value]) : _.orderBy(results, [key], [value]);

                }

                return results;

            }

        },

        methods: {

            resetFilter() {

                this.sort = this.search = '';
                this.star = 0;

            },

            fetchRepositories(params = null) {

                let data = {
                    q: `created:>${this.$parent.date}`
                };

                // Params is used only to send request to Github API for sorting by Stars, however sort will work without this part and <orderBy> watch property calling this method over again
                if (params !== null) data = {...data, ...params}

                this.loading = true;
                axios.get('https://api.github.com/search/repositories', {
                    params: data
                })
                    .then(response => {
                        this.repositories = response.data.items;
                    })
                    .finally(() => this.loading = false);

            }

        }

    }

</script>
