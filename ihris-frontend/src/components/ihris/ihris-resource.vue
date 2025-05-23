<style>
.highlighted {
  background-color: #5396dc !important;
}
</style>
<template>
  <v-container :key="pageKey" class="my-3">
    <v-form
      id="print"
      ref="form"
      v-model="valid"
    >
      <div v-if="!this.edit"
           class="print-section"
      >
        <v-btn
            id="elementToHide"
            color="primary"
            icon
            small
            text
            class="pr-12"
            @click="printPage"
        >
          <v-icon>mdi-printer-outline</v-icon>
            {{$t(`App.hardcoded-texts.Print`)}}
        </v-btn>
      </div>
      <slot :source="source"></slot>
      <v-overlay :value="overlay">
        <v-progress-circular
          color="primary"
          indeterminate
          size="50"
        ></v-progress-circular>
      </v-overlay>

      <v-navigation-drawer id="elementToHide"
        app
        class="primary darken-1 white--text"
        clipped
        permanent
        right
        style="z-index: 3;"
        v-if="showDrawer"
      >
        <v-list class="white--text">
          <v-list-item v-if="edit">
            <v-btn v-if="edit" class="secondary" dark @click="$router.go(-1)">
              <v-icon light>mdi-close-circle-outline</v-icon>
              <span>{{ $t(`App.hardcoded-texts.Cancel`) }}</span>
            </v-btn>
            <v-spacer></v-spacer>
            <template v-if="edit">
              <v-btn v-if="valid" :disabled="!valid" class="success darken-1" dark @click="processFHIR()">
                <v-icon light>mdi-content-save</v-icon>
                <span>{{ $t(`App.hardcoded-texts.Save`) }}</span>
              </v-btn>
              <v-btn v-else class="warning" dark @click="$refs.form.validate()">
                <v-icon light>mdi-content-save</v-icon>
                <span>{{ $t(`App.hardcoded-texts.Save`) }}</span>
              </v-btn>
            </template>
          </v-list-item>
          <v-divider color="white"></v-divider>
          <template v-if="!edit && links && links.length && linksready">
            <v-list-item v-for="(link,idx) in links" :key="link.url">
              <v-btn :key="link.url" :text="!link.button" :to="getLinkUrl(link)" :class="link.linkclass">
                <v-icon v-if="link.icon" light>{{ link.icon }}</v-icon>
                {{ $t(`App.fhir-resources-texts.${linktext[idx]}`) }}
              </v-btn>
            </v-list-item>
          </template>
          <v-divider color="white"></v-divider>
          <v-subheader v-if="sectionMenu" class="white--text"><h2>{{ $t(`App.hardcoded-texts.Sections`) }}</h2>
          </v-subheader>
          <v-list-item
            v-for="section in sectionMenu"
            :id="'#section-' + section.name"
            :key="section.name"
            :class="'#section-' + section.name === path ? 'highlighted' : ''"
            @click="scrollTo(section.name)"
          >
            <v-list-item-content class="white--text" v-if="!edit || !section.secondary">
              <v-list-item-title class="text-uppercase"><h4>{{ $t(`App.fhir-resources-texts.${section.title}`) }}</h4></v-list-item-title>
              <v-list-item-subtitle class="white--text">{{ $t(`App.fhir-resources-texts.${section.desc}`) }}</v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>
        </v-list>

      </v-navigation-drawer>
      <v-card v-if="!this.edit"
        class="mx-auto mb-4 rounded-lg"
        max-width="700"
        outlined
      >
        <v-card-title id="elementToHide" class="primary font-weight-bold caption pa-2 white--text justify-center ">Record History
        </v-card-title>
        <v-card-text  class="my-3">
          <v-row id="elementToHide" class="justify-space-between">
            <v-col cols="4"><span class="font-weight-bold">Record Id:</span></v-col>
            <v-col cols="8">{{ this.$data.orig.id }}</v-col>
          </v-row>
          <v-divider/>
          <v-row class="justify-space-between">
            <v-col cols="4"><span class="font-weight-bold">Last Updated:</span></v-col>
            <v-col cols="8" v-if="$data.orig && $data.orig.meta">
              {{ new Date(this.$data.orig.meta.lastUpdated) }}
            </v-col>
          </v-row>
          <v-divider/>
          <v-row class="justify-space-between">
            <v-col cols="4"><span class="font-weight-bold">Version:</span></v-col>
            <v-col cols="8" v-if="$data.orig && $data.orig.meta">{{ $data.orig.meta.versionId }} of {{ max }}</v-col>
          </v-row>
          <v-divider/>
          <v-row id="elementToHide" class="justify-space-between">
            <v-col cols="4"><span class="font-weight-bold">See older version:</span></v-col>
            <v-col cols="8">
              <v-row>
                <v-col class="px-4" cols="2">
                  <v-text-field
                    v-if="$data.orig.meta"
                    v-model.number="version"
                    :max="max"
                    :rules="[rules.min, rules.max]"
                    :value="$data.orig.meta.versionId"
                    class="mt-0 pt-0"
                    hide-details
                    min="1"
                    single-line
                    style="width: 40px;"
                    type="number"
                  ></v-text-field>
                </v-col>
                <v-col class="px-4">
                  <v-btn
                    :disabled="version>max || version<1"
                    color="primary"
                    icon
                    small
                    @click="changeVersion"
                  >
                    <v-icon>mdi-eye</v-icon>
                  </v-btn>
                </v-col>
              </v-row>
            </v-col>
          </v-row>
          <v-divider/>
        </v-card-text>
      </v-card>
    </v-form>
  </v-container>

</template>

<script>

export default {
  name: "ihris-resource",
  props: ["title", "field", "fhir-id", "fhir-version", "page", "profile", "section-menu", "edit", "links", "mounts", "constraints", "show-drawer"],
  data: function () {
    return {
      fhir: {},
      orig: {},
      valid: true,
      source: {data: {}, path: ""},
      mountedSources: {},
      path: "",
      loading: false,
      overlay: false,
      isEdit: false,
      linktext: [],
      linksready: false,
      advancedValid: true,
      loadingId: false,
      loadingCv: false,
      version: 1,
      max: 1,
      pageKey: 0,
      rules: {
        min: v => v >= 1 || `The Min is 1`,
        max: v => v <= this.max || `The Max is ${this.max}`
      },
    }
  },
  created: function () {
    if (this.fhirId) {
      this.loading = true
      //console.log("getting",this.field,this.fhirId)
      let url = "/fhir/" + this.field + "/" + this.fhirId
      if(this.fhirVersion) {
        url = "/fhir/vRead/" + this.field + "/" + this.fhirId + "/" + this.fhirVersion
        this.version = this.fhirVersion
      }
      fetch(url).then(response => {
        response.json().then(async(data) => {
          if(this.$store.state.user && this.$store.state.user.obj && this.$store.state.user.obj.resource && this.$store.state.user.obj.resource.extension) {
            let location = this.$store.state.user.obj.resource.extension.find((ext) => {
              return ext.url === "http://ihris.org/fhir/StructureDefinition/ihris-user-location"
            })
            if(location) {
              let relatedGrp = data.extension.find((ext) => {
                return ext.url === "http://ihris.org/fhir/StructureDefinition/ihris-related-group"
              })
              if(relatedGrp) {
                let dataLocation = relatedGrp.extension.find((ext) => {
                  return ext.url === "location"
                })
                if(dataLocation) {
                  let canSee = relatedGrp.extension.find((ext) => {
                    return ext.url === "location" && ext.valueString === location.valueReference.reference
                  })
                  if(!canSee) {
                    this.$store.state.message.active = true
                    this.$store.state.message.type = "error"
                    this.$store.state.message.text = "You dont have access to view this record"
                    this.$router.push({path: "/"})
                  }
                }
              }
            }
          }
          //this.$store.commit('setCurrentResource', data)
          this.max = data.meta.versionId
          this.orig = data
          this.source = {data: data, path: this.field}
          await this.setMountedRefs(data)
          this.setLinkText()
          this.linksready = true
          this.loading = false
        }).catch(err => {
          console.log(this.field, this.fhirId, err)
        })
      }).catch(err => {
        console.log(this.field, this.fhirId, err)
      })
    } else if (this.$route.query) {
      let presets = {
        resourceType: this.field
      }
      let hasPresets = false
      for (let path of Object.keys(this.$route.query)) {
        if (path.startsWith(this.field + ".")) {
          hasPresets = true
          let elements = path.substring(this.field.length + 1).split('.')
          let last = elements.pop()
          let current = presets
          for (let element of elements) {
            if (element.includes('[')) {
              try {
                let parts = element.split('[')
                let name = parts[0]
                let idx = parts[1].slice(0, -1)
                if (!current.hasOwnProperty(name)) {
                  current[name] = []
                }
                if (idx) {
                  let next = {}
                  current[name][parseInt(idx)] = next
                  current = next
                } else {
                  let next = {}
                  current[name].push(next)
                  current = next
                }
              } catch (err) {
                console.log("Unable to process", path)

              }
            } else {
              if (!current.hasOwnProperty(element)) {
                current[element] = {}
                current = current[element]
              }
            }
          }
          if (last.includes('[')) {
            try {
              let parts = last.split('[')
              let name = parts[0]
              let idx = parts[1].slice(0, -1)
              if (!current.hasOwnProperty(name)) {
                current[name] = []
              }
              if (idx) {
                current[name][parseInt(idx)] = this.$route.query[path]
              } else {
                current[name].push(this.$route.query[path])
              }

            } catch (err) {
              console.log("Unable to process", path)

            }
          } else {
            current[last] = this.$route.query[path]
          }
        }
      }
      if (hasPresets) {
        this.source = {data: presets, path: this.field}
      }
    }
  },
  computed: {
    hasFhirId: function () {
      if (this.fhirId == '') {
        return false
      } else if (!this.fhirId) {
        return false
      } else {
        return true
      }
    }
    /*
    source: function() {
      return this.$store.state.fhir
    }
    */
  },
  methods: {
    scrollTo(section) {
      document.getElementById(`section-${section}`).scrollIntoView()
    },
    handleScroll() {
      this.hasScrolled = window.top.scrollY >= 100;
      this.sectionMenu.map((data) => {
        let divs = document.getElementById(`section-${data.name}`);
        let sectionTop = divs.offsetTop;
        if (pageYOffset >= sectionTop) {
          this.path = `#section-${data.name}`;
        }
      });
    },
    setMountedRefs() {
      return new Promise((resolve) => {
        if(!this.mounts || this.mounts.length === 0) {
          return resolve()
        }
        let mount = this.mounts.shift()
        let first = mount.fromref.split(".")[0]
        let resource
        if(this.mountedSources[first]) {
          resource = {}
          resource[first] = this.mountedSources[first]
        } else {
          resource = this.source.data
        }
        let content = this.$fhirpath.evaluate(resource, mount.fromref)
        if(content) {
          fetch("/fhir/" + content[0].split("/")[0] + "/" + content[0].split("/")[1]).then((response) => {
            response.json().then(async(data) => {
              this.mountedSources[mount.name] = data
              if(this.mounts.length > 0) {
                await this.setMountedRefs(data)
                resolve()
              } else {
                resolve()
              }
            })
          })
        } else {
          resolve()
        }
      })
    },
    getLinkField: function (field) {
      let content = this.$fhirpath.evaluate(this.source.data, field)
      if (content) {
        return content[0]
      } else {
        return false
      }
    },
    getMountedLinkParameters(url) {
      let urlPortions = url.split("{")
      let params = []
      for(let portion of urlPortions) {
        if(portion.includes("}")) {
         let param = portion.split("}")[0]
         params.push(param)
        }
      }
      return params
    },
    getLinkUrl: function (link) {
      let mountedParams = this.getMountedLinkParameters(link.url)
      if(mountedParams.length > 0) {
        for(let param of mountedParams) {
          let val = this.$fhirpath.evaluate(this.mountedSources, param)
          if(val) {
            if (val[0].includes('/')) {
              let ref = val[0].split('/')
              val[0] = ref[1]
            }
            link.url = link.url.replace("{" + param + "}", val[0])
          }
        }
      }
      let field
      if (link.field) {
        field = this.getLinkField(link.field)
      }
      if (field) {
        if (field.includes('/')) {
          let ref = field.split('/')
          field = ref[1]
        }
        return link.url.replace("FIELD", field)
      } else {
        return link.url
      }
    },
    setLinkText: function () {
      for (let idx in this.links) {
        let link = this.links[idx]
        if (link.text) {
          this.$set(this.linktext, idx, link.text)
        } else if (link.field) {
          let field = this.getLinkField(link.field)
          if (field) {
            this.$fhirutils.lookup(field).then(display => {
              this.$set(this.linktext, idx, display)
            })
          }
        }
      }
    },
    processFHIR: async function () {
      this.$refs.form.validate()
      if (!this.valid) return
      this.advancedValid = true
      this.overlay = true
      this.loading = true

      //const processChildren = function( parent, obj, children ) {
      const processChildren = async (parent, obj, children) => {
        //console.log("called on "+parent)

        for (let child of children) {

          let fullField = parent

          let next = obj

          if (child.field && !child.fieldType /* ignore arrays */) {
            //console.log("working on "+parent+" . "+child.field)
            let field
            if (child.sliceName) {
              if (child.field.startsWith("value[x]")) {
                field = child.field.substring(9)
                fullField += "." + field
              } else {
                field = child.field.replace(":" + child.sliceName, "")
                fullField += "." + field
              }
            } else {
              field = child.field
              fullField += "." + field
            }
            if (child.max !== "1" || child.baseMax !== "1") {
              if (!obj.hasOwnProperty(field)) {
                next[field] = []
              }
            } else {
              next[field] = {}
            }
            //console.log(fullField)
            //console.log(child.max, child.baseMax)
            //console.log(child)
            if (child.hasOwnProperty("value")) {
              //console.log( fullField +"="+ child.value )
              if (Array.isArray(next[field])) {
                next[field].push(child.value)
              } else {
                next[field] = child.value
              }
              next = next[field]
            } else {
              if (Array.isArray(next[field])) {
                let sub = {}
                if (child.profile) {
                  sub.url = child.profile
                } else if (field === "extension" && child.sliceName) {
                  sub.url = child.sliceName
                }
                next[field].push(sub)
                next = sub
              } else {
                next = next[field]
              }
            }
          }

          if (child.$children) {
            try {
              await processChildren(fullField, next, child.$children)
            } catch (err) {
              this.advancedValid = false
              console.log(err)
            }
          }
          if (child.constraints) {
            child.errors = []
            try {
              this.advancedValid = this.advancedValid && await this.$fhirutils.checkConstraints(child.constraints,
                  this.constraints, next, child.errors, this.fhirId)
            } catch (err) {
              this.advancedValid = false
              child.errors.push("An unknown error occurred.")
              console.log(err)
            }
          }

        }

      }

      //console.log(this.field)
      this.fhir = {
        resourceType: this.field,
        meta: {
          profile: [this.profile]
        }
      }
      //console.log(this)
      try {
        await processChildren(this.field, this.fhir, this.$children)
      } catch (err) {
        this.advancedValid = false
        console.log(err)
      }
      if (!this.advancedValid) {
        this.overlay = false
        this.loading = false
        this.$store.commit('setMessage', {type: 'error', text: 'There were errors on the form.'})
        return
      }
      console.log("FINISHED PROCESS AND CHECK.")
      let url = "/fhir/" + this.field
      let opts = {
        method: "POST",
        headers: {
          "Content-Type": "application/fhir+json"
        },
        redirect: 'manual',
      }
      if (this.fhirId) {
        this.fhir.id = this.fhirId
        url += "/" + this.fhirId
        opts.method = "PUT"
      }
      opts.body = JSON.stringify(this.fhir)
      fetch(url, opts).then(response => {
        //console.log(response.headers)
        if (response.status === 201 || response.status === 200) {
          response.json().then(data => {
            this.overlay = false
            this.loading = false

            if (this.fhirId) {
              this.$store.commit("setMessage", {
                type: "success",
                text: "Update successful.",
              });
              setTimeout(() => this.$router.push({name: "resource_view", params: {page: this.page, id: data.id}}), 1000);
            } else {
              this.$router.push({name: "resource_view", params: {page: this.page, id: data.id}})
            }
          })
        }
      })
    },
    changeVersion() {
      fetch("/fhir/vRead" + "/" + this.field + "/" + this.fhirId + "/" + this.version)
          .then((response) => {
            response
                .json()
                .then((data) => {
                  this.orig = data;
                  this.source = {data: data, path: this.field};
                  this.setLinkText();
                  this.loading = false;
                  this.pageKey += 1;
                })
                .catch((err) => {
                  console.log(this.field, this.fhirId, err);
                });
          })
          .catch((err) => {
            console.log(this.field, this.fhirId, err);
          });
    },
    printPage() {
      this.$root.$emit("printPage", true);
      this.$nextTick(() => {
        let contents = document.getElementById("print").innerHTML;
        let stylesHtml = '';
        for (const node of [...document.querySelectorAll('link[rel="stylesheet"], style')]) {
          stylesHtml += node.outerHTML;
        }
        let windowsFrame = document.createElement('iframe');

        windowsFrame.name = "windowsFrame";
        windowsFrame.style.position = "absolute";
        windowsFrame.style.top = "-1000000px";
        document.body.appendChild(windowsFrame);
        let frameDoc = windowsFrame.contentWindow ? windowsFrame.contentWindow : windowsFrame.contentDocument.document ? windowsFrame.contentDocument.document : windowsFrame.contentDocument;
        frameDoc.document.write(`<!DOCTYPE html>
                  <html>
                    <head>
                      ${stylesHtml}
                      <style>
                </style>
                    </head>
                      ${contents}
                    </body>
                  </html>`);
        const sectionLanguage = frameDoc.document.getElementById('section-language');
        if (sectionLanguage) {
          const headerElement = sectionLanguage.querySelector('.v-card__title');
          if (sectionLanguage.textContent.trim().length === headerElement.textContent.trim().length) {
            sectionLanguage.style.display = 'none';
          }
        }

        const elementsToHide = frameDoc.document.querySelectorAll('#section-element-to-hide');
        elementsToHide.forEach(element => {
          const parentSection = element.closest('.ihris-section');
          if (parentSection) {
            parentSection.style.display = 'none';
          }
        });
        frameDoc.document.close();
        setTimeout(function () {
          window.frames["windowsFrame"].focus();
          window.frames["windowsFrame"].print();
          document.body.removeChild(windowsFrame);
        }, 500);
        return false;
      })
    }
  }
}

</script>
<style scoped>
.print-section {
    max-width: 700px;
    margin: 0 auto; /* Center align */
    text-align: right; /* Align text to the right */
}
@media print {
  #elementToHide {
      display: none;
    }
  div, div * {
    font-family: 'Roboto', sans-serif!important;
    color: black!important;
  }
  button {
      display: none;
    }
}

</style>
