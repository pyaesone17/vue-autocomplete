
<template>
  <div class="AutoComplete" @mouseLeave="leave()" style="width:40%;margin-left:30%;position:relative">
   
    <input type="text" v-model="input" 
           @keyup="search"
           @focus="search"
           @keyup.down="down"
           @keydown.up="up"
           @keypress.enter="enter"
           @click="donothing"
           v-bind:class="inputClass"
           class="AutoComplete__Input" 
           v-bind:name="inputName" 
           autocomplete="off" v-bind:value="inputValue">

    <slot></slot>

    <ul class="AutoComplete__List" >
      
        <li v-for="(result,index) in results" 
            @click="insertItem(result, index)"
            @mouseover="hoverItem(result, index)" 
            class="AutoComplete__Item"
            v-bind:class="{ 'active': active == index }"
            v-show="results.length"
        >
        <img v-show="result.image" v-bind:src="result.image"/>
            <span>
                <span v-if="hightlight.status"><span v-html="result.compile"></span></span>
                <span v-else>{{ result.text }}</span>
            </span>
        </li>

        <li class="AutoComplete__Item" v-if="loading">
            Finding Results for <strong>"{{input}}"</strong> ....
        </li>

        <li class="AutoComplete__Item" 
            v-if="!loading && input!=='' && recentSelect===false " 
            v-bind:class="{ 'active': active === results.length }"
            @mouseover="hoverItem(results.length)" 
        >
            <a style="font-size:19px"> <svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1" id="Capa_1" x="0px" y="0px" viewBox="0 0 52.966 52.966"  xml:space="preserve" width="30px" height="25px" style="display:inline-block;float:left;margin-top:1vw;padding-right:15px">
                <path d="M28.983,20h-14c-0.552,0-1,0.448-1,1s0.448,1,1,1h14c0.552,0,1-0.448,1-1S29.535,20,28.983,20z" fill="#565b6b"/>
                <path d="M51.704,51.273L36.845,35.82c3.79-3.801,6.138-9.041,6.138-14.82c0-11.58-9.42-21-21-21s-21,9.42-21,21s9.42,21,21,21   c5.083,0,9.748-1.817,13.384-4.832l14.895,15.491c0.196,0.205,0.458,0.307,0.721,0.307c0.25,0,0.499-0.093,0.693-0.279   C52.074,52.304,52.086,51.671,51.704,51.273z M2.983,21c0-10.477,8.523-19,19-19s19,8.523,19,19s-8.523,19-19,19   S2.983,31.477,2.983,21z" fill="#565b6b"/>
            </svg>
            <span style="font-size:16px">
            See All Results for <strong>"{{input}}"</strong></span></a>
        </li>

    </ul>
  </div>
</template>

<script>
export default {
  props: [
    'remoteUrl',
    'list',
    'inputName',
    'inputValue',
    'inputClass',
    'hightlight',
    'redirect',
    'afterInsert',
    'afterHover',
    'redirectBlank'
  ],
  mounted () {
    var self = this
    window.onclick = function () {
      self.results = []
    }
  },
  data () {
    return {
      input: '',
      old: '',
      active: -1,
      results: [],
      loading: false,
      recentSelect: false
    }
  },
  methods: {
    search (e) {
      if (this.old === this.input) { return }
      if (e.which === 38 || e.which === 40) { return }
      if (e.which === 13) {
        return
      }
      if (this.input.trim() === '') { this.results = []; return false }
      this.loading = true
      this.recentSelect = false
      if (this.remoteUrl == null) {
        this.localSearch()
      } else {
        this.remoteSearch()
      }
      this.old = this.input
      this.active = -1
    },
    up (e) {
      if (e.which === 38) {
        e.preventDefault()
        this.arrowUp(e)
        return
      }
    },
    down (e) {
      if (e.which === 40) {
        this.arrowDown()
        return
      }
    },
    donothing (e) {
      // stop propagation to window.onclicks
      e.stopPropagation()
    },
    localSearch () {
      let blah = []
      let results = this.list.filter((str) => {
        let haystack = str.text
        let needle = this.input.toLowerCase()
        if (haystack.toLowerCase().indexOf(needle) >= 0) {
          return str
        }
      })
      results.forEach((str) => {
        let haystack = str.text
        let needle = this.input
        let string = this.renderHighlight(haystack, needle)
        blah.push(Object.assign({}, str, {text: str.text, compile: string, image: str.image}))
      })
      this.results = blah
      this.loading = false
    },
    remoteSearch () {
      var ajax = new window.XMLHttpRequest()
      ajax.open('GET', this.remoteUrl + '?q=' + this.input, true)
      ajax.onreadystatechange = () => {
        if (ajax.readyState === 4) {
          let blah = []
          var data = JSON.parse(ajax.response)
          data.forEach((str) => {
            blah.push({text: str.text, compile: str.compile, image: str.image})
          })
          console.log(this.results)
          this.results = blah
        }
      }
      ajax.send()
      this.loading = false
    },
    renderHighlight (haystack, needle) {
      let needleStart = haystack.toLowerCase().indexOf(needle.toLowerCase())
      let needleEnd = needle.length + needleStart
      let color = 'none'
      let string = `${haystack.substring(0, needleStart)}<span style=background:${color}>${haystack.substring(needleStart, needleEnd)}</span>${haystack.substring(needleEnd, haystack.length)}`
      return string
    },
    arrowDown () {
      this.active < this.results.length ? this.active += 1 : this.active = -1
    },
    arrowUp () {
      this.active > 0 ? this.active -= 1 : this.active = this.results.length
    },
    enter (e) {
      if (e.keyCode === 13) {
        let activeObject = this.results[this.active]
        this.redirectAction(activeObject)
        this.input = activeObject.text
        this.old = this.input
        this.results = []
        this.recentSelect = true
      }
    },
    redirectAction (activeObject) {
      if (this.redirect === true && Object.hasOwnProperty.call(activeObject, 'redirectUrl')) {
        if (typeof this.redirectBlank !== 'undefined') {
          window.open(activeObject.redirectUrl, '_blank')
          return
        }
        window.location = activeObject.redirectUrl
      }
    },
    hoverItem (val, index) {
      this.active = index
      if (typeof this.afterHover !== 'undefined') {
        this.afterHover(JSON.stringify(val), index)
      }
    },
    insertItem (val, index) {
      this.input = val.text
      this.results = []
      this.recentSelect = true
      if (typeof this.afterInsert !== 'undefined') {
        this.afterInsert(JSON.stringify(val), index)
      }
      this.redirectAction(val)
    }
  }
}
</script>

<style lang="css" scoped>
  .AutoComplete__Input {
    height: 40px;
    width: 100%;
    display: block;
    padding: 15px 15px 15px 10px;
    font-size: 16px;
    box-sizing:border-box;
    -moz-box-sizing:border-box;
    outline: none;
    border: 0px;
    box-shadow: 0px 0px 2px 0px rgb(175, 175, 175);
  }

  .AutoComplete__List{
    margin: 0;
    margin-top: 5px;
    padding: 0;   
    background: white;
    z-index: 99999;
    list-style: none;
    border-radius: 3px;
    text-align: left;
    width: 100%;
    box-shadow:0px 0px 2px 0px rgb(175, 175, 175);
    position: absolute;
  }

  .AutoComplete__Item:last-child{
    border-radius: 0px 0px 5px 5px;
    border-bottom: 1px solid #f1f1f1;    
  }

  .AutoComplete__Item:first-child{
    /*border-top: 1px solid #f1f1f1;*/
  }

  .AutoComplete__Item{
    font-size: 17px;
    min-height: 50px;
    padding: 0px 10px 0 10px; 
    line-height: 50px;
    cursor: pointer;
    font-size: 16px;
  }


  .AutoComplete__Item.active{
    background:#E91E63;
    color: white;
    margin: 0px;
  }
  .AutoComplete__Item  img{
    width: 30px;
    line-height: 50px;
    height: 30px;
    border-radius: 5px;
    display: block;
    float: left;
    margin-right: 5px;
    margin-top: 10px;
  }

</style>