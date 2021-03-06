<template>
    <div class="addresses">
        <div class="card">
            <div class="header">
                <h2>Addresses</h2>
                <template v-show="!loading && assetsLoaded">
                    <div class="bar">
                        <p class="count">{{ totalAddresses.toLocaleString() }} {{totalAddresses | pluralize }} found</p>
                    </div>    
                </template>
            </div>
            <template v-if="loading && !assetsLoaded">
                <v-progress-circular :size="16" :width="2" color="#E84970" indeterminate key="1"></v-progress-circular>
            </template>
            <template v-else>
                <AddressDataTable :addresses="addressesList" :links="true"></AddressDataTable>
            </template>
        </div>
    </div>
</template>

<script lang="ts">
import { Vue, Component } from "vue-property-decorator";
import api from "@/axios";
import Address from "@/js/Address";
import { IAddress, IAddressData, IBalance_X } from '@/js/IAddress';
import Big from "big.js";
import AddressDataTable from '@/components/Address/AddressDataTable.vue';

@Component({
    components: {
        AddressDataTable
    },
    filters: {
        pluralize(val: number): string {
            return val === 0
                ? `addresses`
                : val > 1
                ? `addresses`
                : `address`;
        }
    }
})

export default class Addresses extends Vue {
    loading: boolean = true;
    addressesList: IAddress[] = [];
    totalAddresses: number = 0;

    async created() {
        this.loading = false;
        // TODO: support service for multiple chains
        api.get("/x/addresses").then(res => {
            this.totalAddresses = res.data.count            
            let addresses: IAddressData = res.data.addresses;

            // TODO: unique addresses or sort in API
            // let addressesMap: {[key:string]: IAddressData} = {};
            // for (let i = 0; i < addresses.length; i++) {
            //     let addressID = addresses[i].address;
            //     if (addressesMap[addressID]) {
            //         console.log("redundant Address", addressID);
            //     } else {
            //         addressesMap[addressID] = addresses[i];
            //     }
            // }
            // let sorted = Object.values(addressesMap).map((addressData: IAddressData) => {
            
            let sorted = Object.values(addresses).map((addressData: IAddressData) => {
                let address: IAddress = {
                    address: addressData.address,
                    publicKey: addressData.publicKey,
                    // P-Chain AVAX balance
                    AVAX_balance: Big(0),
                    P_unlocked: Big(0),
                    P_lockedStakeable: Big(0),
                    P_lockedNotStakeable: Big(0),
                    P_staked: Big(0),
                    P_utxoIDs: [],
                    // X-Chain AVAX balance
                    X_unlocked: Big(0),
                    X_locked: Big(0),
                    // X-Chain Assets
                    totalTransactionCount: 0,
                    totalUtxoCount: 0,
                    assets: [],
                }

                if (this.assetsMap) {
                    address = new Address(addressData, this.assetsMap);
                }
                
                return address;
            });
            sorted.sort((a: IAddress, b: IAddress) => {
                let diff = a.X_unlocked.minus(b.X_unlocked);
                if (diff.gt(0)) return -1;
                else if (diff.lt(0)) return 1;
                else return 0;
            });
            this.addressesList = sorted;
        });
    }

    get assetsMap() {
        return this.$store.state.assets;
    }
}
</script>

<style scoped lang="scss">

.addresses {
    font-size: 12px;
}

.header {
    padding-bottom: 20px;
    margin-bottom: 10px;
}

.bar {
    display: flex;
    align-items: center;
    > p {
        flex-grow: 1;
    }
}

@include smOnly {
    .table_headers {
        display: none;
    }
}
</style>
