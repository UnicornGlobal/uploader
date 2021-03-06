<template>
    <div class="container" :style="containerStyle">
        <modal v-if="showCropper" @close="reset">
            <h5 slot="header">Edit</h5>
            <div slot="body">
                <cropper ref="cropper"
                         classname="cropper"
                         :src="cropperImage"
                         :stencil-component="stencil"
                         :stencil-props="{aspectRatio: this.cropping.aspectRatio || 1}"
                         :restrictions="imageDimensions"
                />
                <br>
                <div style="text-align: right">
                    <button :style="buttonStyle" @click.prevent="finishCropping">Save</button>
                </div>
            </div>
        </modal>
        <div v-if="disabled" class="mask"></div>
        <div class="wrapper">
            <div ref="preview" class="preview" :style="previewStyle" @click.prevent="pickFile()">
                <template v-if="!previewImage">
                    <slot name="preview" @click.prevent="pickFile()">
                        <div
                                v-if="!(previewName || placeholderName)"
                                class="placeholder"
                                :style="placeholderStyle">
                        </div>
                        <div
                                v-if="!previewName"
                                class="preview-name placeholder-name">
                            {{placeholderName}}
                        </div>
                        <div
                                v-if="previewName"
                                ref="preview-name"
                                class="preview-name">
                            {{previewName}}
                        </div>
                    </slot>
                </template>
                <slot name="loading" v-if="uploading">
                    <loader
                            fill="pink">
                    </loader>
                </slot>
                <a @click.prevent="reset()"
                   v-if="hasFile && !instantUpload"
                   :style="resetStyle"
                   class="reset-button">
                </a>
            </div>
            <button
                    v-if="isValid && url && !instantUpload"
                    @click.prevent="uploadFile()"
                    class="picker"
                    :style="buttonDisabledStyle"
                    :disabled="uploading">
                Save
            </button>
            <button
                    v-else-if="!buttonless"
                    @click.prevent="pickFile()"
                    class="picker"
                    :style="buttonStyle">
                Select
            </button>
            <input @change="onChange" ref="input" type="file" name="file" v-validate="rules">
        </div>
        <span v-if="errors.has('file')" class="validation-error">
      {{errors.first('file')}}
    </span>
    </div>
</template>

<script>
    import Loader from './Loader.vue'
    import placeholder from './assets/preview.svg'
    import Modal from './Modal.vue'
    import {Cropper, RectangleStencil, CircleStencil} from 'vue-advanced-cropper'

    export default {
        components: {
            Loader,
            placeholder,
            Modal,
            Cropper
        },
        props: {
            cropping: {
                required: false,
                default: false
            },
            validation: {
                type: Object,
                required: false,
                default: () => {
                    return {}
                }
            },
            instantUpload: {
                type: Boolean,
                required: false,
                default: false
            },
            buttonless: {
                type: Boolean,
                required: false,
                default: false
            },
            name: {
                type: String,
                required: true
            },
            url: {
                type: String,
                required: false
            },
            fieldName: {
                type: String,
                required: false,
                default: 'document'
            },
            disabled: {
                type: Boolean,
                required: false,
                default: false
            },
            placeholder: {
                required: false,
                default: false
            },
            previewRadius: {
                required: false,
                default: 0
            },
            preview: {
                required: false,
                default: () => {
                    return `url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 22 22"><defs id="defs3051"><style type="text/css" id="current-color-scheme">.ColorScheme-Text { color:#4d4d4d; }</style></defs><path style="fill:currentColor;fill-opacity:1;stroke:none" d="M 3 3 L 3 19 L 19 19 L 19 3 L 3 3 z M 4 4 L 18 4 L 18 18 L 4 18 L 4 4 z M 7.5 5 C 6.115 5 5 6.115 5 7.5 C 5 8.885 6.115 10 7.5 10 C 8.885 10 10 8.885 10 7.5 C 10 6.115 8.885 5 7.5 5 z M 7.5 6 C 8.331 6 9 6.669 9 7.5 C 9 8.331 8.331 9 7.5 9 C 6.669 9 6 8.331 6 7.5 C 6 6.669 6.669 6 7.5 6 z M 13.994141 10 L 10.648438 13.34375 L 9.3066406 12 L 9.3007812 12.007812 L 9.2929688 12 L 5 16.292969 L 5.7070312 17 L 9.2988281 13.408203 L 9.9414062 14.050781 L 10.648438 14.757812 L 11.355469 14.050781 L 14 11.40625 L 16.292969 13.699219 L 17 12.992188 L 14.007812 10 L 14 10.007812 L 13.994141 10 z " class="ColorScheme-Text" /></svg>')`
                }
            },
            resetImage: {
                required: false,
                default: () => {
                    return `url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20"><path stroke="red" stroke-width="3.4" d="m2.4,2.4 15.2,15.2m0-15.2-15.2,15.2"/></svg>')`
                }
            },
            styles: {
                required: false,
                default: () => {
                    return {
                        box: {
                            width: '230px',
                            height: '210px',
                            margin: '1em'
                        },
                        preview: {
                            width: '210px',
                            height: '200px',
                            margin: '1em',
                            border: '0'
                        },
                        button: {
                            colour: 'green',
                            text: 'white',
                            disabled: 'grey',
                            width: '210px',
                            margin: '0 1em 1em'
                        }
                    }
                }
            }
        },
        data() {
            return {
                showCropper: false,
                uploading: false,
                files: null,
                isValid: false,
                cropperImage: '',
                previewImage: '',
                placeholderName: '',
                oldimage: null,
                stencil: RectangleStencil
            }
        },
        created() {
            if (!this.$http) {
                throw 'Vue Axios is required on `this.$http`'
            }

            if (
                !this.$validator ||
                !this.$validator.validateAll
            ) {
                throw 'Vee Validate is required on `this.$validate`'
            }

            if (this.cropping && this.cropping.shape === 'round') {
                this.stencil = CircleStencil
            }
        },
        watch: {
            previewImage(image) {
                this.$refs['preview'].style.backgroundImage = `url(${image})`
            }
        },
        computed: {
            containerStyle() {
                return `width: ${this.styles.box.width}; height: ${this.styles.box.height}; margin: ${this.styles.box.margin};`
            },
            previewStyle() {
                return `width: ${this.styles.preview.width}; height: ${this.styles.preview.height}; margin: ${this.styles.preview.margin}; background-image: ${this.preview}; border-radius: ${this.previewRadius}px; border: ${this.preview ? this.styles.preview.border : ""}`
            },
            buttonStyle() {
                return `background-color: ${this.styles.button.colour}; width: ${this.styles.button.width}; margin: ${this.styles.button.margin}; color: ${this.styles.button.text}; text-align: center;`
            },
            buttonDisabledStyle() {
                return `background-color: ${this.styles.button.disabled}; width: ${this.styles.button.width}; margin: ${this.styles.button.margin}; color: ${this.styles.button.text};`
            },
            resetStyle() {
                return `background-image: ${this.resetImage}`
            },
            placeholderStyle() {
                if (this.placeholder) {
                    return `background-image: ${this.placeholderUrl}`
                }

                return `background-image: ${this.preview}`
            },
            hasFile() {
                return !!(this.files && this.files.length)
            },
            rules() {
                if (this.validation && this.validation.rules) {
                    return this.validation.rules
                }
                return {}
            },
            previewName() {
                if (!this.hasFile) {
                    return ''
                }

                if (this.files[0].type.startsWith('image')) {
                    return ''
                }
                return this.files[0].name.split('/').pop()
            },
            placeholderUrl() {
                if (!this.placeholder) {
                    return false
                }

                let name = this.placeholder.split('/').pop()
                let extension = name.split('.').pop()

                if (['png', 'jpg', 'jpeg'].includes(extension)) {
                    return `url(${this.placeholder})`
                }

                this.placeholderName = name
                return false
            }
        },
        methods: {
            imageDimensions(_) {
                return {
                    minHeight: this.cropping.minHeight || 400,
                    minWidth: this.cropping.minWidth || 400,
                    maxHeight: this.cropping.maxHeight || 1000,
                    maxWidth: this.cropping.maxWidth || 1000
                }
            },
            pickFile() {
                this.files = null
                this.$refs['input'].click()
            },
            onChange(event) {
                return this.$validator.validateAll().then((valid) => {
                    if (!valid) {
                        this.uploading = false
                    }
                    this.isValid = valid
                    this.files = Array.from(event.target.files)
                    this.$emit('input', this.files)
                    this.readFile()
                })
            },
            readFile() {
                if (!this.hasFile || !this.files[0].type.startsWith('image')) {
                    return
                }

                this.uploading = true
                this.oldimage = this.previewImage
                this.previewImage = ''
                let reader = new window.FileReader()
                reader.addEventListener('load', () => {
                    //if cropping is enabled, launch cropper
                    if (this.cropping && this.cropping.enabled) {
                        this.launchCropper(reader.result)
                    } else {
                        this.previewImage = reader.result

                        if (this.instantUpload) {
                            this.uploadFile()
                        } else {
                            this.uploading = false
                        }
                    }
                })
                reader.readAsDataURL(this.files[0])
            },
            launchCropper(imageData) {
                this.showCropper = true;
                this.cropperImage = imageData
            },
            finishCropping() {
                const {canvas} = this.$refs.cropper.getResult()
                this.files[0] = this.fileFromBase64(canvas.toDataURL(), this.files[0].name, this.files[0].type)
                this.uploadFile()
            },
            fileFromBase64(base64File, fileName, mimeType) {
                const i = base64File.indexOf('base64,');
                const data = base64File.slice(i + 7);
                const bytes = atob(data)
                const buffer = new ArrayBuffer(bytes.length)

                const ia = new Uint8Array(buffer);
                for (let i = 0; i < bytes.length; i += 1) {
                    ia[i] = bytes.charCodeAt(i);
                }

                return new File([ia], fileName, {
                    type: mimeType,
                });
            },
            reset() {
                this.previewImage = ''
                this.cropperImage = ''
                this.showCropper = false
                this.$refs['input'].value = null
                this.files = null
                this.isValid = false
                this.errors.clear()
                this.$nextTick(() => {
                    this.uploading = false
                })
            },
            uploadFile() {
                return this.$validator.validateAll().then((valid) => {
                    if (valid) {
                        this.uploading = true
                        return this.doUpload()
                    } else {
                        this.uploading = false
                    }
                })
            },
            doUpload() {
                let formData = new FormData()
                let file = this.files[0]
                formData.append(this.fieldName, file, file.name)
                formData.append('title', this.name)
                this.$emit('progress', 1)
                return this.$http.post(this.url, formData, {
                    headers: {'Content-Type': 'multipart/form-data'},
                    onUploadProgress: (event) => {
                        this.$emit('progress', (event.total / event.loaded) * 100)
                    }
                }).then((response) => {
                    this.$emit('uploaded', response.data)
                    this.reset()
                    return response
                }).catch((error) => {
                    this.previewImage = this.oldimage
                    this.$emit('failed', error)
                    this.reset()
                    return error
                })
            }
        }
    }
</script>

<style lang="scss" scoped>
    .container {
        position: relative;
        margin: 1em;

        .mask {
            position: absolute;
            width: 100%;
            height: 100%;
            background-color: #d6d6d6;
            z-index: 100;
            opacity: 0.8;
        }
    }

    .wrapper {
        display: flex;
        flex-direction: column;
    }

    .preview {
        cursor: pointer;
        position: relative;
        display: flex;
        width: 100%;
        align-self: center;
        background-color: white;
        background-size: cover;
        background-position: center;

        .reset-button {
            position: absolute;
            right: 0;
            width: 30px;
            height: 30px;
            background-position: center;
            background-size: cover;
            background-repeat: no-repeat;
        }

        .preview-name {
            padding-left: 5px;
            padding-right: 5px;
            text-align: center;
            align-self: center;
            margin-left: auto;
            margin-right: auto;
            overflow-wrap: break-word;
            word-wrap: break-word;
            word-break: break-all;
            hyphens: auto;
        }

        .placeholder {
            position: absolute;
            width: 100%;
            height: 100%;
            background-size: cover;
            background-position: center;
        }
    }

    input {
        display: none;
    }

    .picker {
        margin-top: auto;
        align-self: center;
    }
</style>
