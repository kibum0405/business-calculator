<template>
    <v-container>
        <!-- 공통 헤더 -->
        <v-toolbar flat class="mb-3" style="background-color: white;">
            <v-card-title>디마 장부</v-card-title>
            <v-spacer></v-spacer>
            <v-btn icon @click="undoItem">
                <v-icon>mdi-undo</v-icon>
            </v-btn>
            <v-btn icon @click="redoItem">
                <v-icon>mdi-redo</v-icon>
            </v-btn>
        </v-toolbar>
        <v-row>
            <v-col cols="6">
                <!-- 재고 데이터 테이블 -->
                <v-data-table
                    class="elevation-1 mb-3 inventory-table"
                    :headers="headers"
                    :items="filteredItems"
                    :items-per-page="5"
                    :search="search"
                >
                    <template v-slot:top>
                        <v-toolbar flat>
                            <v-card-title>재고 : {{ totalInventory }}</v-card-title>
                            <v-text-field
                                v-model="search"
                                label="검색"
                                single-line
                                hide-details
                                variant="outlined"
                                density="compact"
                            ></v-text-field>
                        </v-toolbar>
                    </template>
                    <template v-slot:item="{ item }">
                        <tr>
                            <td>{{ item.name }}</td>
                            <td>{{ formatPrice(item.purchasePrice, item.unit) }}</td>
                            <td>{{ item.quantity }}</td>
                            <td>{{ formatPrice(item.purchasePrice * item.quantity, item.unit) }}</td>
                            <td>{{ item.purchaseDate }}</td>
                            <td>
                                <v-btn v-for="action in actions" :key="action.icon" @click="action.method(item)" density="comfortable" variant="text" icon>
                                    <v-icon small>{{ action.icon }}</v-icon>
                                </v-btn>
                            </td>
                        </tr>
                    </template>
                </v-data-table>
            </v-col>
        
            <v-col cols="6">
                <!-- 판매 데이터 테이블 -->
                <v-data-table
                    class="elevation-1 mb-3 sales-table"
                    :headers="saleHeaders"
                    :items="filteredSaleItems"
                    :items-per-page="5"
                    :search="saleSearch"
                >
                    <template v-slot:top>
                        <v-toolbar flat>
                            <v-card-title>순익 : {{ totalSales }}</v-card-title>
                            <v-text-field
                                v-model="saleSearch"
                                label="판매 검색"
                                single-line
                                hide-details
                                variant="outlined"
                                density="compact"
                            ></v-text-field>
                        </v-toolbar>
                    </template>
                    <template v-slot:item="{ item }">
                        <tr :style="{ backgroundColor: getRowColor(item) }">
                            <td>{{ item.name }}</td>
                            <td>{{ item.salePrice }}</td>
                            <td>{{ item.quantity }}</td>
                            <td>{{ item.saleDate }}</td> <!-- 추가된 부분 -->
                            <td>{{ item.profit }}</td>
                            <td>{{ item.profitRate }}</td>
                        </tr>
                    </template>
                </v-data-table>
            </v-col>
        </v-row>
        <!-- 입력 폼 컴포넌트 -->
        <purchaseItem v-if="!isSelling"
            ref="purchaseItemForm"
            :item="editedItem"
            :is-editing="isEditing"
            @submit="saveItem"
            @cancel="cancelEdit"
        />
        <saleItem v-else
            ref="saleItemForm"
            :item="editedItem"
            @submit="saveSale"
            @cancel="cancelSale"
        />
    </v-container>
</template>

<script>
import purchaseItem from './purchaseItem.vue';
import saleItem from './saleItem.vue';


export default {
    name: 'mainPage',
    components: {
        purchaseItem,
        saleItem,
    },
    data() {
        return {
            search: '',
            isEditing: false,
            editedIndex: -1,
            editedItem: {
                name: '',
                purchasePrice: '',
                purchaseDate: '',
                quantity: '',
            },
            defaultItem: {
                name: '',
                purchasePrice: '',
                purchaseDate: '',
                quantity: '',
            },
            isSelling: false,
            headers: [
                { title: '이름', key: 'name' },
                { title: '가격', key: 'purchasePrice' },
                { title: '수량', key: 'quantity' },
                { title: '총 가격', key: 'totalPrice' },
                { title: '날짜', key: 'purchaseDate' },
                { title: '작업', key: 'actions', sortable: false },
            ],
            actions: [
                { icon: 'mdi-pencil', method: this.editItem },
                { icon: 'mdi-cash', method: this.sellItem },
                { icon: 'mdi-delete', method: this.deleteItem },
            ],
            saleHeaders: [
                { title: '이름', key: 'name' },
                { title: '가격', key: 'salePrice' },
                { title: '수량', key: 'quantity' },
                { title: '날짜', key: 'saleDate' }, // 추가된 부분
                { title: '순이익', key: 'profit' },
                { title: '수익율', key: 'profitRate' },
            ],
            items: JSON.parse(localStorage.getItem('inventoryItems')) || [],
            saleItems: JSON.parse(localStorage.getItem('saleItems')) || [],
            undoHistory: [],
            redoHistory: [],
            saleSearch: '',
        };
    },
    mounted() {
        // 페이지 로드 시 localStorage에서 데이터 불러오기
        const savedItems = localStorage.getItem('inventoryItems');
        const savedSaleItems = localStorage.getItem('saleItems');
        if (savedItems) {
            this.items = JSON.parse(savedItems);
        }
        if (savedSaleItems) {
            this.saleItems = JSON.parse(savedSaleItems);
        }
    },
    computed: {
        totalInventory() {
            const totals = this.items.reduce((acc, item) => {
                const unit = item.unit || '원';
                if (!acc[unit]) acc[unit] = 0;
                acc[unit] += item.purchasePrice * item.quantity;
                return acc;
            }, {});

            const formattedTotals = ['원', 't', 'm'].map(unit => {
                const value = totals[unit] || 0;
                return `${this.formatPrice(value, unit)}`;
            });
            return `( ${formattedTotals.join(' / ')} )`;
        },
        totalSales() {
            const totals = this.saleItems.reduce((acc, item) => {
                const unit = item.unit || '원';
                if (!acc[unit]) acc[unit] = 0;
                // 순이익을 사용합니다
                acc[unit] += this.parsePriceString(item.profit);
                return acc;
            }, {});

            const formattedTotals = ['원', 't', 'm'].map(unit => {
                const value = totals[unit] || 0;
                return this.formatPrice(value, unit);
            });
            return `( ${formattedTotals.join(' / ')} )`;
        },
        filteredItems() {
            return this.items.filter(item => {
                return Object.keys(item).some(key =>
                    String(item[key]).toLowerCase().includes(this.search.toLowerCase())
                );
            });
        },
        filteredSaleItems() {
            return this.saleItems.filter(item => {
                return Object.keys(item).some(key =>
                    String(item[key]).toLowerCase().includes(this.saleSearch.toLowerCase())
                );
            });
        }
    },
    watch: {
        items: {
            handler() {
                this.saveToLocalStorage();
            },
            deep: true
        },
        saleItems: {
            handler() {
                this.saveToLocalStorage();
            },
            deep: true
        }
    },
    methods: {
        parsePriceString(priceString) {
            // "1,000,000 원" 형식의 문자열에서 숫자만 추출
            return parseFloat(priceString.replace(/[^0-9.-]+/g,""));
        },
        getRowColor(item) {
            // profit 값에서 숫자만 추출
            const profit = parseFloat(item.profit.replace(/[^0-9.-]+/g, ""));
            return profit >= 0 ? '#E1F5FE' : '#FBE9E7';
        },
        saveToLocalStorage() {
            localStorage.setItem('inventoryItems', JSON.stringify(this.items));
            localStorage.setItem('saleItems', JSON.stringify(this.saleItems));
        },
        addItem(newItem) {
            const itemToAdd = { ...newItem };
            if (!itemToAdd.purchaseDate) {
                itemToAdd.purchaseDate = this.getTodayDate();
            }
            
            const existingItemIndex = this.items.findIndex(item => item.name === itemToAdd.name);
            
            if (existingItemIndex !== -1) {
                // 이름이 같은 항목이 있을 경우
                const existingItem = this.items[existingItemIndex];
                this.undoHistory.push({action: 'edit', item: {...existingItem}});
                this.redoHistory = []; // 새로운 변경사항이 있으면 redo 히스토리 초기화
                
                const newQuantity = parseInt(existingItem.quantity) + parseInt(itemToAdd.quantity);
                const newTotalPrice = (parseFloat(existingItem.purchasePrice) * parseInt(existingItem.quantity)) +
                                      (parseFloat(itemToAdd.purchasePrice) * parseInt(itemToAdd.quantity));
                const newAveragePrice = newTotalPrice / newQuantity;
                
                // 모든 필드 업데이트
                existingItem.quantity = newQuantity;
                existingItem.purchasePrice = newAveragePrice.toFixed(2);
                existingItem.purchaseDate = itemToAdd.purchaseDate; // 날짜 업데이트
            } else {
                // 새 항목 추가
                this.undoHistory.push({action: 'add', item: {...itemToAdd}});
                this.redoHistory = []; // 새로운 변경사항이 있으면 redo 히스토리 초기화
                this.items.push(itemToAdd);
            }
            this.saveToLocalStorage();
            this.closeEdit();
        },
        editItem(item) {
            this.editedIndex = this.items.indexOf(item);
            this.editedItem = Object.assign({}, item, { unit: item.unit || '원' });
            this.isEditing = true;
        },
        deleteItem(item) {
            const index = this.items.indexOf(item);
            if (confirm('이 항목을 삭제하시겠습니까?')) {
                this.undoHistory.push({action: 'delete', item: {...this.items[index]}});
                this.redoHistory = [];
                this.items.splice(index, 1);
            }
            this.saveToLocalStorage();
        },
        saveItem(item) {
            if (this.isEditing) {
                // 수정
                const index = this.editedIndex;
                if (index !== -1) {
                    this.undoHistory.push({action: 'edit', item: {...this.items[index]}});
                    this.redoHistory = [];
                    Object.assign(this.items[index], item);
                }
            } else {
                // 새 항목 추가
                this.addItem(item);
            }
            this.saveToLocalStorage();
            this.closeEdit();
        },
        closeEdit() {
            this.isEditing = false;
            this.isSelling = false;
            this.editedItem = Object.assign({}, this.defaultItem);
            this.editedIndex = -1;
            this.$nextTick(() => {
                if (this.$refs.purchaseItemForm) {
                    this.$refs.purchaseItemForm.resetForm();
                }
                if (this.$refs.saleItemForm) {
                    this.$refs.saleItemForm.resetForm();
                }
            });
        },
        cancelEdit() {
            this.closeEdit();
        },
        getTodayDate() {
            const today = new Date();
            return today.toISOString().substr(0, 10); // 'YYYY-MM-DD' 형식으로 반환
        },
        undoItem() {
            if (this.undoHistory.length > 0) {
                const lastAction = this.undoHistory.pop();
                if (lastAction.action === 'edit') {
                    const index = this.items.findIndex(item => item.name === lastAction.item.name);
                    if (index !== -1) {
                        this.redoHistory.push({action: 'edit', item: {...this.items[index]}});
                        this.items[index] = {...lastAction.item};
                    }
                } else if (lastAction.action === 'add') {
                    const index = this.items.findIndex(item => item.name === lastAction.item.name);
                    if (index !== -1) {
                        this.redoHistory.push({action: 'delete', item: {...this.items[index]}});
                        this.items.splice(index, 1);
                    }
                } else if (lastAction.action === 'delete') {
                    this.redoHistory.push({action: 'add', item: {...lastAction.item}});
                    this.items.push({...lastAction.item});
                } else if (lastAction.action === 'sell') {
                    const soldItem = this.saleItems.pop();
                    const existingItemIndex = this.items.findIndex(item => item.name === soldItem.name);
                    if (existingItemIndex !== -1) {
                        this.items[existingItemIndex].quantity = parseInt(this.items[existingItemIndex].quantity) + parseInt(soldItem.quantity);
                    } else {
                        this.items.push({...lastAction.item, quantity: soldItem.quantity});
                    }
                    this.redoHistory.push({action: 'unsell', item: soldItem});
                }
                this.saveToLocalStorage();
            }
        },
        redoItem() {
            if (this.redoHistory.length > 0) {
                const lastAction = this.redoHistory.pop();
                if (lastAction.action === 'edit') {
                    const index = this.items.findIndex(item => item.name === lastAction.item.name);
                    if (index !== -1) {
                        this.undoHistory.push({action: 'edit', item: {...this.items[index]}});
                        this.items[index] = {...lastAction.item};
                    }
                } else if (lastAction.action === 'delete') {
                    const index = this.items.findIndex(item => item.name === lastAction.item.name);
                    if (index !== -1) {
                        this.undoHistory.push({action: 'add', item: {...this.items[index]}});
                        this.items.splice(index, 1);
                    }
                } else if (lastAction.action === 'add') {
                    this.undoHistory.push({action: 'delete', item: {...lastAction.item}});
                    this.items.push({...lastAction.item});
                } else if (lastAction.action === 'unsell') {
                    const item = lastAction.item;
                    const index = this.items.findIndex(i => i.name === item.name);
                    if (index !== -1) {
                        this.items[index].quantity = Math.max(0, parseInt(this.items[index].quantity) - parseInt(item.quantity));
                        if (this.items[index].quantity === 0) {
                            this.items.splice(index, 1);
                        }
                    }
                    this.saleItems.push(item);
                    this.undoHistory.push({action: 'sell', item: {...this.items[index]}});
                }
                this.saveToLocalStorage();
            }
        },
        sellItem(item) {
            this.editedIndex = this.items.indexOf(item);
            this.editedItem = Object.assign({}, item, { 
                salePrice: '', 
                saleDate: this.getTodayDate(),
                unit: item.unit || '원'
            });
            this.isSelling = true;
        },
        saveSale(item) {
            const index = this.editedIndex;
            if (index !== -1) {
                const originalItem = {...this.items[index]};
                this.undoHistory.push({action: 'sell', item: originalItem});
                this.redoHistory = [];
                
                // 재고 수량 업데이트
                this.items[index].quantity -= item.quantity;
                if (this.items[index].quantity < 0) {
                this.items[index].quantity = 0;
                }
                
                // 판매 데이터 추가
                const profit = item.salePrice * item.quantity - originalItem.purchasePrice * item.quantity;
                const profitRate = ((item.salePrice - originalItem.purchasePrice) / originalItem.purchasePrice * 100).toFixed(2);
                
                const saleItem = {
                name: item.name,
                salePrice: this.formatPrice(item.salePrice, item.unit),
                quantity: item.quantity,
                saleDate: item.saleDate,
                profit: this.formatPrice(profit, item.unit),
                profitRate: `${profitRate}%`
                };
                this.saleItems.push(saleItem);
            }
            this.saveToLocalStorage();
            this.closeEdit();
        },
        cancelSale() {
            this.closeEdit();
        },
        formatPrice(price, unit) {
            return `${Number(price).toLocaleString()} ${unit}`;
        },
    },
};
</script>

<style>
table td,
table th {
    padding: 0px 4px 0px 4px !important;
}
.v-container {
    max-width:1680px;
}
.v-toolbar__content {
    height:48px !important;
    padding:10px;
}

.v-table__wrapper {
    height:350px;
    overflow: auto;
}

.inventory-table {
    border-radius: 8px !important;
}

.sales-table {
    border-radius: 8px !important;
}

.inventory-table .v-toolbar {
    background-color:#B3E5FC;
    border-top-left-radius: 8px;
    border-top-right-radius: 8px;
}

.sales-table .v-toolbar {
    background-color:#B2DFDB;
    border-top-left-radius: 8px;
    border-top-right-radius: 8px;
}

.v-text-field .v-input__details {
    padding-inline : 0px;
    
}

.v-input__details {
    padding-top: 0px !important;
    min-height: 0px !important;
}

.input-text-field {
    margin-bottom:10px;
}

.input-text-field input {
    padding:8px !important;
}
</style>