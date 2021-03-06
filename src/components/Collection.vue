<template>
  <div>
    <div :class="'blip-container collection'" v-touch:swipe.left="swipeLeftHandler" v-touch:swipe.right="swipeRightHandler" v-if="document.itemType === 'application/vnd.lime.document-select+json'">
      <div :class="'slideshow-container ' + position" :id="id">
        <div class="slideshow-list">
          <div class="slideshow-track">
            <div v-for="(item, index) in items" v-bind:key="index" @dblclick="editCard(item)">
              <document-select :on-cancel="cancel" :editing="item.editing" :length="95" class="slide-item" :position="position" :on-selected="onSelected" :on-open-link="onOpenLink" :document="item" :deletable="deletable"
                :editable="editable" :on-save="collectionSave" :style="styleObject" :on-deleted="deleteItem" />
            </div>
            <div v-if="newDocumentSelect.editing">
              <document-select :on-cancel="cancel" :editing="newDocumentSelect.editing" :style="styleObject" :length="95" class="slide-item" :position="position" :on-selected="onSelected" :on-open-link="onOpenLink"
                :document="newDocumentSelect" :editable="editable" :on-save="addToCollection" :on-deleted="deleteItem" />
            </div>
            <div v-if="editable" @click="newDocumentSelect.editing = true">
              <div :class="'collection-editable slide-item'" :style="styleObject">
                <img :src="plusSvg" style="position: absolute; top: 50%; left: 50%; width: 50px; height: 50px; margin-top: -25px; margin-left: -25px" />
              </div>
            </div>
          </div>
        </div>

        <a class="prev" v-if="showPrev" @click="plusSlides(-1)">&#10094;</a>
        <a class="next" v-if="showNext" @click="plusSlides(1)">&#10095;</a>
      </div>

      <div :class="'notification ' + position" v-if="date">
        {{ date }}
      </div>
    </div>

    <div v-else-if="document.itemType === 'application/vnd.lime.container+json'">
      <div v-for="(item, index) in document.items" v-bind:key="index">
          <blip-card :position="position" :date="date" :on-selected="onSelected" :on-open-link="onOpenLink" :document="{ type: item.type, content: item.value }" :editable="editable" />
      </div>
    </div>
    <div v-else>
      <div v-for="(item, index) in document.items" v-bind:key="index">
        <blip-card :position="position" :date="date" :on-selected="onSelected" :on-open-link="onOpenLink" :document="{ type: document.itemType, content: item }" :editable="editable" />
      </div>
    </div>
  </div>
</template>

<script>
import { guid } from '../utils/misc'
import { default as base } from '../mixins/baseComponent.js'
import cloneDeep from 'lodash/cloneDeep'

let newCollection = {
  header: {
    type: 'application/vnd.lime.media-link+json',
    value: { title: '', text: '', type: '', uri: '', aspectRatio: '' }
  },
  options: [],
  editing: false
}

export default {
  name: 'collection',
  mixins: [base],
  props: {
    length: {
      type: Number,
      default: 532
    },
    initWith: {
      type: Number,
      default: 2
    },
    onSelected: {
      type: Function
    },
    onOpenLink: {
      type: Function
    }
  },
  computed: {
    showPrev: function() {
      return this.slideIndex !== 1
    },
    showNext: function() {
      let length = this.editable ? this.items.length + 1 : this.items.length
      return this.slideIndex !== length
    }
  },
  data: function() {
    return {
      id: undefined,
      slideIndex: undefined,
      width: undefined,
      elementsWidth: undefined,
      elementsLength: undefined,
      items: undefined,
      styleObject: undefined,
      newDocumentSelect: undefined
    }
  },
  watch: {
    document: function() {
      this.items = this.document.items.map(function(x, i) {
        return {
          ...x,
          id: i
        }
      })
    }
  },
  mounted: function() {
    this.mounted()
  },
  methods: {
    editCard: function(item) {
      if (this.editable) {
        item.editing = true
      }
    },
    mounted: function() {
      if (
        this.document.itemType === 'application/vnd.lime.document-select+json'
      ) {
        let element = this.$el
        if (!element) return

        let listElement = element.querySelector('.slideshow-list')
        this.width = parseInt(
          window
            .getComputedStyle(listElement)
            .width.toString()
            .replace('px', '')
        )

        let elements = element.querySelectorAll('.slide-item')
        this.elementsLength = elements.length

        for (let i = 0; i < this.elementsLength; i++) {
          if (this.width <= 400) {
            this.elementsWidth = this.width - 50
            elements[i].style.width = this.width - 50 + 'px'
          } else {
            this.elementsWidth = this.width / this.initWith
            elements[i].style.width = this.elementsWidth + 'px'
          }
        }

        this.styleObject['width'] = this.elementsWidth + 'px'
        this.showSlides(this.slideIndex)
      }
    },
    init: function() {
      this.id = guid()
      this.slideIndex = 1
      this.width = 0
      this.elementsWidth = 0
      this.elementsLength = 0
      this.items = this.document.items.map(function(x, i) {
        return {
          ...x,
          id: i,
          editing: false
        }
      })
      this.styleObject = {
        'margin-top': '20px'
      }
      this.newDocumentSelect = cloneDeep(newCollection)
      this.mounted()
    },
    swipeLeftHandler: function() {
      if (this.showNext) {
        this.plusSlides(1)
      }
    },
    swipeRightHandler: function() {
      if (this.showPrev) {
        this.plusSlides(-1)
      }
    },
    deleteItem: function(document) {
      this.items = this.items.filter((x) => x !== document)

      if (this.items.length === 0) {
        this.trash({
          ...this.document,
          items: this.items
        })
      } else {
        this.save({
          ...this.document,
          items: this.items
        })
      }
    },
    addToCollection: function(document) {
      this.items.push(document)
      this.newDocumentSelect = cloneDeep(newCollection)
      document.editing = false

      this.save({
        ...this.document,
        items: this.items
      })
    },
    collectionSave: function(document) {
      let items = this.items.filter((x) => x.id !== document.id)
      items.splice(document.id, 0, document)
      document.editing = false

      this.save({
        ...this.document,
        items: items
      })
      this.isEditing = true
    },
    plusSlides: function(n) {
      this.showSlides((this.slideIndex += n))
    },
    showSlides: function(n) {
      var element = this.$el
      let trackElement = element.querySelector('.slideshow-track')

      if (n === 1) {
        trackElement.setAttribute('style', 'transform: translate3d(0px, 0px, 0px); -webkit-transform: translate3d(0px, 0px, 0px);')
      } else {
        let margin = this.elementsWidth === this.width ? -10 : 10
        const data =
          'translate3d(' +
          (((this.elementsWidth + 10) * (n - 1) - margin) * -1 + 10) +
          'px, 0px, 0px)'
        trackElement.setAttribute('style', `transform: ${data}; -webkit-transform: ${data};`)
      }
    }
  }
}
</script>

<style lang="scss">
@import '../styles/variables.scss';

.collection-editable {
  height: 100%;
  border: 1px dashed $vue-time;
  position: relative;
  cursor: pointer;
}

.collection {
  .slideshow-container {
    margin: auto;

    .slideshow-list {
      overflow: hidden;
      margin: 0;
      padding: 0px 30px;
    }

    .slideshow-track {
      transition: all 0.8s ease;
      opacity: 1;
      width: 30000px;
      transform: translate3d(0px, 0px, 0px);
      display: flex;
      top: 0;
      left: 0;
    }

    .bubble {
      width: 100%;
      height: 100%;
      max-width: 100%;
    }

    .slide-item {
      float: left;
      height: 100%;
      min-height: 1px;
      margin-right: 10px;
    }

    a {
      text-decoration: none;
    }
  }

  .prev,
  .next {
    cursor: pointer;
    position: absolute;
    top: 50%;
    width: auto;
    margin-top: -22px;
    padding: 8px 16px;
    opacity: 0.3;
    color: $vue-light-blip;
    font-weight: bold;
    opacity: 0.3;
    font-size: 18px;
    transition: 0.6s ease;
    border-radius: 5px 0 0 5px;
    box-shadow: -2px 2px 20px 0 rgba(51, 60, 74, 0.4);
    background-color: #ffffff;
  }

  .prev {
    left: 0;
  }

  /* Position the "next button" to the right */
  .next {
    right: 18px;
    border-radius: 3px 0 0 3px;
  }

  /* On hover, add a black background color with a little bit see-through */
  .prev:hover,
  .next:hover {
    opacity: 1;
  }

  .fade {
    -webkit-animation-name: fade;
    -webkit-animation-duration: 1.5s;
    animation-name: fade;
    animation-duration: 1.5s;
  }

  @-webkit-keyframes fade {
    from {
      opacity: 0.4;
    }
    to {
      opacity: 1;
    }
  }

  @keyframes fade {
    from {
      opacity: 0.4;
    }
    to {
      opacity: 1;
    }
  }
}
</style>
