<template>
    <DialogEx persistent v-model="opened" width="780" @action:ok="onOK" @action:cancel="onCancel">
        <v-card>
            <v-card-title class="subheading">Create Wallet</v-card-title>
            <v-card-text style="height:390px" class="pt-0">
                <v-stepper v-if="step<4" class="elevation-0" v-model="step" style="height:100%;">
                    <v-stepper-header class="elevation-0">
                        <v-stepper-step :complete="step > 1" step="1"/>
                        <v-divider/>
                        <v-stepper-step :complete="step > 2" step="2"/>
                        <v-divider/>
                        <v-stepper-step :complete="step > 3" step="3"/>
                    </v-stepper-header>
                    <div class="title font-weight-light pl-4">{{stepTitles[step-1]}}</div>
                    <v-stepper-items>
                        <v-stepper-content step="1">
                            <v-form ref="form">
                                <v-text-field
                                    v-focus
                                    validate-on-blur
                                    label="Wallet name"
                                    v-model="name"
                                    maxlength="20"
                                    :counter="20"
                                    :rules="nameRules"
                                ></v-text-field>
                                <v-text-field
                                    validate-on-blur
                                    label="Password"
                                    type="password"
                                    v-model="password"
                                    maxlength="20"
                                    :rules="passwordRules"
                                ></v-text-field>
                                <v-text-field
                                    validate-on-blur
                                    label="Repeat password"
                                    type="password"
                                    v-model="repeatedPassword"
                                    maxlength="20"
                                    :rules="repeatedPasswordRules"
                                ></v-text-field>
                            </v-form>
                        </v-stepper-content>
                        <v-stepper-content step="2">
                            <MnemonicWords :words="words"/>
                        </v-stepper-content>
                        <v-stepper-content step="3">
                            <WordPuzzle
                                :words="words"
                                @click.native="puzzleChange"
                                v-model="puzzleSovled"
                            />
                            <div
                                class="error--text pt-2"
                                v-if="!puzzleSovled && showPuzzleError"
                            >Please double check the mnemonic words on your paper</div>
                        </v-stepper-content>
                    </v-stepper-items>
                </v-stepper>
                <v-layout v-else column fill-height>
                    <v-layout v-if="wallet" column align-center fill-height>
                        <span class="headline font-weight-light">Congratulations</span>
                        <div class="py-3">
                            <v-icon small color="success">mdi-check-decagram</v-icon>
                            <span>This is your new wallet!</span>
                        </div>
                        <WalletCard
                            flat
                            class="card-border-thin wallet-card"
                            style="border-radius:9px;width:170px;"
                            :wallet="wallet"
                        />
                        <QRCode :size="80" class="mt-3">{{wallet.address | checksum}}</QRCode>
                    </v-layout>
                    <v-layout
                        v-else-if="error"
                        column
                        align-center
                        justify-center
                        fill-height
                    >{{`${error.name}: ${error.message}`}}</v-layout>
                    <v-layout v-else column align-center justify-center fill-height>
                        <p>Processing... a monent</p>
                        <v-progress-linear color="success" indeterminate></v-progress-linear>
                    </v-layout>
                </v-layout>
            </v-card-text>
            <v-divider/>
            <v-card-actions>
                <v-btn v-show="step<4" ref="abort" flat small @click="onAbort" tabindex="2">Abort</v-btn>
                <v-spacer/>
                <v-btn
                    small
                    v-show="step<4 && step>1"
                    class="secondary"
                    dark
                    flat
                    @click="onBack"
                    tabindex="1"
                >Back</v-btn>
                <v-btn
                    ref="next"
                    :disabled="processing"
                    :flat="!processing"
                    small
                    class="primary"
                    @click="onNext"
                >{{(step > 3 && !processing) ? 'Done' : 'Next'}}</v-btn>
            </v-card-actions>
        </v-card>
    </DialogEx>
</template>
<script lang="ts">
import { Vue, Component, Mixins, Watch } from 'vue-property-decorator'
import DialogHelper from '@/renderer/mixins/dialog-helper'
import { cry } from 'thor-devkit'

@Component
export default class CreateWalletDialog extends Mixins(
    class extends DialogHelper<void, entities.Wallet | null> { }
) {
    opened = false
    step = 1
    stepTitles = [
        'Please fill the fields below to create your wallet.',
        'Mnemonic words are used to recover your wallet. Please write down and keep them in a secure place.',
        'To make sure the mnemonic words you backed up are correct, please select them in order to verify.'
    ]
    name = ''
    password = ''
    repeatedPassword = ''
    words = generateWords()
    showPuzzleError = false
    puzzleSovled = false
    wallet: entities.Wallet | null = null
    error: Error | null = null

    get processing() {
        return this.step > 3 && !this.wallet && !this.error
    }
    readonly nameRules = [
        (val: string) =>
            (!!val && !!val.trim()) ||
            'Create a name that is simple enough to remember'
    ]
    readonly passwordRules = [
        (val: string) =>
            (!!val && val.length >= 6) || 'Requires at least 6 characters'
    ]
    get repeatedPasswordRules() {
        return [(val: string) => val === this.password || 'Password mismatch']
    }

    mounted() {
        this.opened = true
    }

    close(result: entities.Wallet | null) {
        this.$resolve(result)
        this.opened = false
    }

    puzzleChange() {
        this.showPuzzleError = false
    }

    onNext() {
        if (this.processing) {
            return
        }

        if (this.step === 1) {
            if (!(this.$refs.form as any).validate()) {
                return
            }
        } else if (this.step === 2) {
        } else if (this.step === 3) {
            if (!this.puzzleSovled && !ENV.devMode) {
                this.showPuzzleError = true
                return
            }
            this.encryptAndSave()
        } else {
            this.close(this.wallet)
            return
        }
        this.step++
    }

    onAbort() {
        if (this.step > 3 || this.processing) {
            return
        }
        this.close(null)
    }

    onBack() {
        this.showPuzzleError = false
        if (this.step < 2 || this.processing) {
            return
        }
        this.step--
    }

    onOK() {
        const el = (this.$refs.next as Vue).$el
        el.focus()
        el.click()
    }

    onCancel() {
        const el = (this.$refs.abort as Vue).$el
        el.focus()
        el.click()
    }

    async encryptAndSave() {
        try {
            const privateKey = cry.mnemonic.derivePrivateKey(this.words)
            const ks = await cry.Keystore.encrypt(privateKey, this.password)
            const entity = {
                name: this.name.trim(),
                address: '0x' + ks.address,
                keystore: ks,
                createdTime: Date.now()
            }
            await BDB.wallets.add(entity)
            this.wallet = entity
        } catch (err) {
            this.error = err
        }
    }
}

function generateWords() {
    for (; ;) {
        // to avoid duplicated words
        const words = cry.mnemonic.generate()
        const map: { [i: string]: any } = []
        if (
            words.every(w => {
                if (map[w]) {
                    return false
                }
                map[w] = 1
                return true
            })
        ) {
            return words
        }
    }
}
</script>
<style scoped> 
.theme--dark .wallet-card {
    background-color: #383838;
}
</style>
