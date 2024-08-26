<template>
    <v-card class="pa-4 sale-card-box">
        <v-form ref="form" v-model="valid">
            <template v-for="field in inputFields" :key="field.key">
                <v-row v-if="field.key === 'salePrice'" align="center" no-gutters>
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
                    :disabled="isSelling && field.key === 'name' ? true : false"
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
                판매 완료
            </v-btn>
            <v-btn color="secondary" @click="cancelSale">
                취소
            </v-btn>
        </v-row>
    </v-card>
</template>

<script>
export default {
    name: 'saleItem',
    props: {
        item: {
            type: Object,
            default: () => ({
                name: '',
                salePrice: '',
                saleDate: '',
                quantity: '',
                unit: '원',
            }),
        },
        isSelling: Boolean
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
                    key: 'salePrice',
                    label: '가격',
                    rules: [v => !!v || ''],
                    required: true,
                    type: 'number'
                },
                {
                    key: 'quantity',
                    label: `수량 (재고: ${this.item.quantity})`,
                    rules: [v => !!v || '', v => v <= this.item.quantity || '재고보다 많은 수량을 판매할 수 없습니다'],
                    required: true,
                    type: 'number'
                },
                {
                    key: 'saleDate',
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
        cancelSale() {
            this.$emit('cancel');
        },
        resetForm() {
            this.$refs.form.reset();
            this.localItem = { ...this.item, unit: this.item.unit || '원' };
        },
    },
    watch: {
        item: {
            handler(newItem) {
                this.localItem = { ...newItem, unit: newItem.unit || '원' };
                // 수량 필드의 라벨 업데이트
                const quantityField = this.inputFields.find(field => field.key === 'quantity');
                if (quantityField) {
                    quantityField.label = `수량 (max: ${newItem.quantity})`;
                }
            },
            immediate: true,
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