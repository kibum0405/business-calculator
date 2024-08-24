<template>
    <v-card class="pa-4 purchase-card-box">
        <v-form ref="form" v-model="valid">
            <template v-for="field in inputFields" :key="field.key">
                <v-row v-if="field.key === 'purchasePrice'" align="center" no-gutters>
                    <v-col cols="8">
                        <v-text-field class="input-text-field"
                            v-model="localItem[field.key]"
                            :rules="field.rules"
                            :label="field.label"
                            :required="field.required"
                            :type="field.type"
                            variant="outlined"
                            density="compact"
                        ></v-text-field>
                    </v-col>
                    <v-col cols="4" class="pl-4" style="margin-top:-10px;">
                        <v-select
                            v-model="localItem.unit"
                            :items="units"
                            label="단위"
                            variant="outlined"
                            density="compact"
                        ></v-select>
                    </v-col>
                </v-row>
                <v-text-field v-else class="input-text-field"
                    v-model="localItem[field.key]"
                    :rules="field.rules"
                    :label="field.label"
                    :required="field.required"
                    :type="field.type"
                    variant="outlined"
                    density="compact"
                ></v-text-field>
            </template>
        </v-form>
        <v-row class="ma-0 pa-0 edit-btn-box">
            <v-spacer></v-spacer>
            <v-btn :disabled="!valid" color="primary" @click="submitForm">
                {{ isEditing ? '항목 수정' : '항목 추가' }}
            </v-btn>
            <v-btn v-if="isEditing" color="secondary" @click="cancelEdit">
                취소
            </v-btn>
        </v-row>
    </v-card>
</template>

<script>
export default {
    name: 'purchaseItem',
    props: {
        item: {
            type: Object,
            default: () => ({
                name: '',
                purchasePrice: '',
                purchaseDate: '',
                quantity: '',
            }),
        },
        isEditing: {
            type: Boolean,
            default: false,
        },
    },
    data() {
        return {
            valid: false,
            localItem: { ...this.item, unit: '원' },
            units: ['원', 't', 'm'],
            inputFields: [
                {
                    key: 'name',
                    label: '이름',
                    rules: [v => !!v || ''],
                    required: true,
                    type: 'text'
                },
                {
                    key: 'purchasePrice',
                    label: '가격',
                    rules: [v => !!v || ''],
                    required: true,
                    type: 'number'
                },
                {
                    key: 'quantity',
                    label: '수량',
                    rules: [v => !!v || ''],
                    required: true,
                    type: 'number'
                },
                {
                    key: 'purchaseDate',
                    label: '날짜',
                    rules: [],
                    required: false,
                    type: 'date'
                }
            ]
        };
    },
    methods: {
        submitForm() {
            if (this.$refs.form.validate()) {
                this.$emit('submit', this.localItem);
            }
        },
        cancelEdit() {
            this.$emit('cancel');
        },
        resetForm() {
            this.$refs.form.reset();
            this.localItem = {
                name: '',
                purchasePrice: '',
                purchaseDate: '',
                quantity: '',
            };
        },
    },
    watch: {
        item: {
            handler(newItem) {
                this.localItem = { ...newItem };
            },
            deep: true,
        },
    },
};
</script>
<style scoped>
.edit-btn-box .v-btn {
    margin-left: 10px;
}

</style>