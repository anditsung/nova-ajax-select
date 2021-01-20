<template>
    <default-field :field="field" :errors="errors">
        <template slot="field">
            <select v-model="value" class="w-full form-control form-select" :disabled="disabled">
                <option :value="null" v-if="loaded && options.length">{{ __('Choose an option') }}</option>
                <option :value="null" v-if="loaded && options.length == 0">{{ __('No Results') }}</option>
                <option
                    :key="option.value"
                    :value="option.value"
                    v-for="option in options">
                    {{ option.display }}
                </option>
            </select>
        </template>
    </default-field>
</template>

<script>
    import { FormField, HandlesValidationErrors } from 'laravel-nova'

    export default {
        mixins: [FormField, HandlesValidationErrors],

        props: ['resourceName', 'resourceId', 'field'],

        data() {
            return {
                options: [],
                loaded: false,
                parentValue: [],
            }
        },

        mounted() {

            this.watchedComponents.forEach(component => {

                let attribute = 'value'

                if (component.field.component === 'belongs-to-field') {

                    attribute = 'selectedResource'

                }

                component.$watch(attribute, (value) => {

                    this.parentValue[component.field.attribute] = (value && attribute == 'selectedResource') ? value.value : value

                    this.updateOptions()

                }, { immediate: true })

            })
        },

        methods: {
            setInitialValue() {
                this.value = this.field.value || ''
            },

            fill(formData) {
                formData.append(this.field.attribute, this.value || '')
            },

            isWatchingComponent(component) {
                if (Array.isArray(this.field.parent_attribute)) {
                    return component.field !== undefined
                        && this.field.parent_attribute.find(attribute => attribute == component.field.attribute)
                }

                return component.field !== undefined
                    && component.field.attribute == this.field.parent_attribute
            },

            shouldUpdate() {
                if (Array.isArray(this.field.parent_attribute)) {
                    var parentValueValid = true
                    this.field.parent_attribute.forEach(attribute => {
                        if (this.parentValue[attribute] == undefined
                            || this.parentValue[attribute] == null
                            || this.parentValue[attribute] == ''
                        ) {
                            parentValueValid = false
                        }
                    })
                    if (parentValueValid) {
                        return true
                    }
                }

                if (this.parentValue[this.field.parent_attribute] != null && this.parentValue[this.field.parent_attribute] != '') {
                    return true
                }

                return false
            },

            endpoint() {
                var endpoint = this.field.endpoint
                    .replace('{resource-name}', this.resourceName)
                    .replace('{resource-id}', this.resourceId ? this.resourceId : '')

                if (Array.isArray(this.field.parent_attribute)) {
                    this.field.parent_attribute.forEach(attribute => {
                        endpoint = endpoint
                            .replace('{' + attribute + '}', this.parentValue[attribute] ? this.parentValue[attribute] : '')
                    })
                }
                else {
                    endpoint = endpoint
                        .replace('{'+ this.field.parent_attribute +'}', this.parentValue[this.field.parent_attribute] ? this.parentValue[this.field.parent_attribute] : '')
                }

                return endpoint
            },

            async updateOptions() {
                this.options = []
                this.loaded = false

                if (this.shouldUpdate()) {

                    Nova
                        .request()
                        .get(this.endpoint())
                        .then(response => {
                            this.options = response.data
                            this.loaded = true

                            let optionValueExists = false;
                            this.options.forEach(option => {
                                if(option.value == this.value) {
                                    optionValueExists = true;
                                }
                            })

                            if(optionValueExists == false) {
                                this.value = null;
                            }
                        })
                } else {
                    this.loaded = false
                }

            },

        },

        computed: {
            watchedComponents() {
                if (!this.field.parent_attribute) {
                    return []
                }

                return this.$parent.$children.filter(component => {
                    return this.isWatchingComponent(component)
                })
            },

            disabled() {
                return this.loaded == false && this.options.length == 0
            },
        },
    }
</script>
