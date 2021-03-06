<template>
    <section class="request-signature">

        <section class="floating-header">
            <figure class="identity-name">{{identity().name}}</figure>
            <figure class="account-authority">{{`${identity().account.name}@${identity().account.authority}`}}</figure>
            <figure class="switches">
                <figure class="switch"
                        v-for="displayType in displayTypes"
                        v-on:click="setDisplayType(displayType)"
                        :class="{'active':selectedDisplayType === displayType}">
                    {{displayType}}
                </figure>
            </figure>
        </section>

        <section class="prompt-body">

            <section class="partitioned"
                     v-if="selectedDisplayType === displayTypes.PROPS"
                     v-for="message in messages">
                <section class="partition">

                    <!-- Contract Name -->
                    <figure class="label">contract</figure>
                    <figure class="value big">{{message.code}}</figure>

                    <!-- Contract Action -->
                    <figure class="label">action</figure>
                    <figure class="value big">{{message.type}}</figure>

                    <section class="key-value" v-if="prompt.data.requiredFields.length">
                        <figure class="key">requires</figure>
                        <figure class="value">
                            {{prompt.data.requiredFields.join(', ')}}
                        </figure>

                        <section v-if="viableLocations.length && selectedLocation">
                            <sel :selected="selectedLocation"
                                 :options="viableLocations"
                                 :parser="location => location.name"
                                 v-on:changed="changed => bind(changed, 'selectedLocation')"></sel>

                            <section style="margin-top:10px;" v-for="(value, key) in selectedLocation" v-if="requiredFields.includes(key)">
                                <figure class="label">{{key}}</figure>
                                <figure class="value">{{typeof value === 'object' ? value.name : value}}</figure>
                            </section>
                        </section>
                    </section>

                </section>
                <section class="partition">

                    <section v-for="(value, key) in sortedMessageData(message.data)">
                        <figure class="label">{{key}}</figure>
                        <figure class="value" :class="{'red':value !== '' && !isNaN(value)}">{{value}}</figure>
                    </section>
                </section>
            </section>

            <section class="json-display" v-else>
                <pre><code>{{messages}}</code></pre>
            </section>
        </section>

        <section class="prompt-footer">

            <section class="whitelist">
                <figure class="header">
                    Do you want to whitelist this contract action?
                </figure>
                <figure class="sub-header">
                    <section class="checkbox">
                        <cin :tag="(whitelisted) ? 'fa-check' : ''" :checkbox="true" v-on:untagged="toggleWhitelist"></cin>
                    </section>
                    You can whitelist this action so that next time you wont have to manually authorize this.
                    If any of the properties of the transaction changes the permission will become invalid. This is based on the messages themselves,
                    so different bundles of messages will pass if they include some or all of the same messages.
                    <br><br>
                    <b>This also includes required personal data, and permissions are not revoked when you update your Identity's information.</b>
                    However, if you have multiple locations it will always provide a prompt regardless of the permission.
                </figure>
            </section>

            <section class="actions">
                <btn text="Deny" v-on:clicked="denied"></btn>
                <btn text="Accept" margined="true" is-blue="true" v-on:clicked="accepted"></btn>
            </section>

        </section>

    </section>
</template>

<script>
    import { mapActions, mapGetters, mapState } from 'vuex'
    import * as Actions from '../store/constants';
    import {RouteNames} from '../vue/Routing'
    import AlertMsg from '../models/alerts/AlertMsg'
    import IdentityService from '../services/IdentityService'
    import NotificationService from '../services/NotificationService'
    import Identity from '../models/Identity'
    import {LocationFields, PersonalFields} from '../models/Identity'
    import ObjectHelpers from '../util/ObjectHelpers'

    const displayTypes = {
        JSON:'json',
        PROPS:'properties'
    };

    export default {
        data(){ return {
            selectedDisplayType:displayTypes.PROPS,
            displayTypes,
            whitelisted:false,

            viableLocations:[],
            selectedLocation:null,

            returnedFields:{},
        }},
        computed: {
            ...mapState([
                'scatter',
                'prompt'
            ]),
            ...mapGetters([
                'messages',
                'identities',
                'requiredFields'
            ])
        },
        mounted(){
            const hasAllRequiredFields = this.identity().hasRequiredFields(this.requiredFields);

            if(!hasAllRequiredFields){
                this[Actions.PUSH_ALERT](AlertMsg.NoIdentityWithProperties(this.requiredFields)).then(closed => {
                    //TODO: Better error handling
                    this.prompt.responder(null);
                    NotificationService.close();
                });
            }

            if(this.requiresLocationDetails()){
                const requiredLocationFields = Object.keys(LocationFields).filter(field => this.requiredFields.includes(field));
                this.viableLocations = this.identity().locations.filter(location => location.findFields(requiredLocationFields).length === requiredLocationFields.length);
                this.selectedLocation = this.viableLocations.find(location => location.isDefault) || this.viableLocations[0];
                this.returnedFields = this.identity().clone();
                this.returnedFields.location = this.selectedLocation;
            }
        },
        methods: {
            setDisplayType(type){ this.selectedDisplayType = type; },

            bind(changed, dotNotation) {
                let props = dotNotation.split(".");
                const lastKey = props.pop();
                props.reduce((obj,key)=> obj[key], this)[lastKey] = changed;
            },

            requiresLocationDetails(){
                return !!this.requiredFields.find(field => Object.keys(LocationFields).includes(field));
            },

            /***
             * Sorts the message by importance.
             * Numeric values come first
             * @param data
             * @returns {*|{}}
             */
            sortedMessageData(data){
                return Object.keys(data)
                    .sort((keyA,keyB) => {
                        const valueA = data[keyA];
                        const valueB = data[keyB];
                        if(!valueA || valueA === '' || !valueB || valueB === '') return -1;
                        if(isNaN(valueA) || isNaN(valueB)) return 1;
                        return 0;
                    })
                    .reduce((acc, key) => {
                        acc[key] = data[key];
                        return acc;
                    }, {})
            },
            identity(){
                return this.scatter.keychain.findIdentity(this.prompt.data.identityHash);
            },
            accepted(){

                const returnedFields = Identity.asReturnedFields(this.requiredFields, this.returnedFields, this.selectedLocation);

                this.prompt.responder({accepted:true, whitelisted:this.whitelisted, returnedFields});
                NotificationService.close();
            },
            denied(){
                this.prompt.responder(null);
                NotificationService.close();
            },
            isCurrencyContract(){ return !!this.messages.find(message => message.code === 'eos') },
            toggleWhitelist(){
                const flip = () => this.whitelisted = !this.whitelisted;

                if(this.isCurrencyContract() && !this.whitelisted) {
                    const alert = AlertMsg.AreYouSure('You Are About To Whitelist A Currency Contract', ['Request', 'Signature', 'Whitelist'],
                        'Whitelisting currency based contracts is dangerous, and should never be done. There are specific cases where this is okay, ' +
                        'but unless you are absolutely sure this is one of them, you should not be whitelisting this contract action. ' +
                        'Are you sure you still want to whitelist this?')
                    this[Actions.PUSH_ALERT](alert).then(res => {
                        if(!res || !res.hasOwnProperty('accepted')) return false;
                        flip();
                    });
                }
                else flip();

            },
            ...mapActions([
                Actions.UPDATE_STORED_SCATTER,
                Actions.PUSH_ALERT,
                Actions.BACKUP_SCATTER_ON_BLOCKCHAIN
            ])
        }
    }
</script>

<style lang="scss">
    .request-signature {
        font-family: 'Open Sans', sans-serif;

        .floating-header {
            position:absolute;
            top:40px;
            right:50px;
            text-align:right;

            .identity-name {
                font-family: 'Raleway', sans-serif;
                font-size:14px;
                font-weight:bold;
                color:#4f4f4f;
                margin-bottom:2px;
            }

            .account-authority {
                color:#54a7fc;
                font-size:9px;
                margin-bottom:5px;
                font-weight: bold;
            }

            .switches {

                .switch {
                    cursor: pointer;
                    height:15px;
                    line-height:13px;
                    font-size:9px;
                    float:right;
                    padding:0 5px;
                    background:transparent;
                    border:1px solid #4f4f4f;
                    border-radius:4px;
                    color:#4f4f4f;
                    margin-left:5px;

                    &:hover, &.active {
                        background:#4f4f4f;
                        color:#fff;
                    }
                }
            }
        }


        .prompt-body {
            height:287px;
            overflow:hidden;

            .key-value {
                padding:20px;
                border:2px dashed rgba(0,0,0,0.05);
                border-radius: 4px;
                margin-bottom:20px;
                margin-top:20px;

                .key {
                    font-size:9px;
                    color:#bbbbbb;
                }

                .value {
                    font-size:12px;
                    color:#707070;
                    font-weight:900;
                    font-style: italic;
                }
            }

            .description {
                margin-top:5px;
                font-size:9px;
                color:#b8b8b8;

                b { color:#707070; }
            }

            .partitioned {
                overflow:hidden;
                height:287px;
                width:100%;

                .partition {
                    width:50%;
                    float:left;
                    height:287px;
                    overflow-y:auto;
                    position: relative;
                    display:block;
                    padding:40px 50px;

                    &:first-child { text-align: left; }
                    &:last-child { text-align: right; border-left:1px solid rgba(0,0,0,0.02); }
                }
            }

            .label {
                font-size:9px;
                color:#bbbbbb;
            }

            .value {
                color:#707070;
                font-size:12px;
                line-height:12px;
                font-style: italic;
                font-weight: 600;
                margin-bottom:15px;

                &.big {
                    font-size:22px;
                    line-height:22px;
                    font-weight:200;
                }

                &.red {
                    font-size:20px;
                    line-height:20px;
                    padding:5px 10px 5px 5px;
                    color:#fff;
                    background:#ff0d0c;
                    display:inline-block;
                    border-radius:4px;
                }
            }

            .json-display {
                padding:20px;
                font-size:14px;
                height:287px;
                overflow-y:auto;
            }
        }

        .prompt-footer {
            height:150px;
            padding:20px;
            background:#fff;
            overflow:hidden;

            .whitelist {
                width:calc(100% - 150px);
                padding-right:20px;
                float:left;

                .checkbox {
                    width:56px;
                    float:left;
                    margin-right:15px;
                }

                &:not(:last-child){
                    border-bottom:1px solid #eaeaea;
                }

                .header {
                    color:#cecece;
                    font-size:11px;
                    padding-bottom:5px;
                    margin-top:-5px;
                    margin-bottom:10px;
                    border-bottom:1px solid #eaeaea;
                }

                .sub-header {
                    color:#aeaeae;
                    font-size:9px;
                    margin-bottom:20px;
                }
            }

            .actions {
                width:150px;
                float:left;
            }
        }
    }

</style>