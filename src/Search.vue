<script lang="ts" setup>
import { ref, onMounted, computed } from "vue";
import { useData, withBase, useRouter } from "vitepress";

// @ts-ignore
import Index from "./module/index.js";

//TODO: delete deprecate code
const VPData = useData();

// @ts-ignore
const locale = VPData.localeIndex || VPData.localePath;

const metaKey = ref();
const open = ref<Boolean>(false);
const searchTerm = ref<string | null>();
const origin = ref<string>("");
const input = ref();
const INDEX_DATA = ref();
const PREVIEW_LOOKUP = ref();
const Options = ref<Options>();
const searchIndex = ref();
const buttonLabel = ref("Search");
const placeholder = ref("Search docs");
const focused = ref(0)
const modal = ref<HTMLElement>()
  const router = useRouter()
interface Options {
  previewLength: number;
  buttonLabel: string;
  placeholder: string;
}

const result = computed(() => {
  if (searchTerm.value) {
    var searchResults = searchIndex.value.search(searchTerm.value, { enrich: true })

    var search = [] as any[];

    for (var i = 0; i < searchResults.length; i++) {
      var id = searchResults[i];
      var item = PREVIEW_LOOKUP.value[id];

      var title = item["t"];
      var preview = item["p"];
      var link = item["l"];
      var anchor = item["a"];
      link = link.split(" ").join("-");
      search.push({id: i, link, title, preview, anchor });
    }
    return search as any[];
  }
});

const GroupBy = (array: any, func: Function) => {
  if (!array || !array.length) return [];

  return array.reduce((acc: any, value: any) => {
    // Group initialization
    if (!acc[func(value)]) {
      acc[func(value)] = [];
    }

    // Grouping
    acc[func(value)].push(value);

    return acc;
  }, {});
};

const openSearch = () => {
  setTimeout(() => {
    if (input.value) input.value.focus();
  }, 100);
  cleanSearch();
  open.value = true;
};

onMounted(async () => {
  const data = await import("virtual:search-data");
  INDEX_DATA.value = data.default.INDEX_DATA;
  PREVIEW_LOOKUP.value = data.default.PREVIEW_LOOKUP;
  Options.value = data.default.Options;
  origin.value = window.location.origin + withBase(locale.value === 'root' ? '/' : locale.value);
  buttonLabel.value = Options.value?.buttonLabel || buttonLabel.value;
  placeholder.value = Options.value?.placeholder || placeholder.value;

  var document = new Index(Options.value);

  document.import("reg", INDEX_DATA.value.reg)
  document.import("cfg", INDEX_DATA.value.cfg)
  document.import("map", INDEX_DATA.value.map)
  document.import("ctx", INDEX_DATA.value.ctx)

  searchIndex.value = document

  metaKey.value.innerHTML = /(Mac|iPhone|iPod|iPad)/i.test(navigator.platform)
    ? "⌘"
    : "Ctrl";

  const handleSearchHotKey = (e: KeyboardEvent) => {
    if (e.key === "k" && (e.ctrlKey || e.metaKey)) {
      e.preventDefault();
      openSearch();
    }
    if (e.key === "Escape"){
      if (searchTerm.value?.length == 0 && open.value)
        open.value = false
    }
  };

  window.addEventListener("keydown", handleSearchHotKey);
  

});

const groupedResults = computed(() => GroupBy(result.value, (x:any) =>
                  x.link.split('/').slice(0, -1).join('-')
                ))

const linksOrder = computed(() => Object.values(groupedResults.value).flat().map(i => i.id))

function cleanSearch() {
  open.value = false;
  searchTerm.value = "";
}

const handleNavigation = (e: KeyboardEvent) => {  
  if(!document.getElementsByClassName("search-group")[0])
    return

  if(e.key == 'ArrowUp' || e.key == 'ArrowDown'){
    const currentIndex = linksOrder.value.indexOf(focused.value)
    if(e.key == 'ArrowUp'){
      const previousIndex = currentIndex == 0 ? linksOrder.value.length - 1: currentIndex - 1 
      focused.value =  linksOrder.value[previousIndex]
    }
    if(e.key == 'ArrowDown'){
      const nextIndex = currentIndex > linksOrder.value.length - 2 ? 0 : currentIndex + 1
      focused.value = linksOrder.value[nextIndex]
    }
    
    if(currentIndex < 5 ){
        const el = modal.value!.getElementsByClassName("VPPluginSearch-search-group")[0]
        el?.scrollIntoView(true)
    }else
       document.getElementById(linksOrder.value[focused.value])?.scrollIntoView(true)
       
  }
  if(e.key == "Enter"){
    router.go(VPData.site.value.base + modal.value!.getElementsByClassName('link-focused')[0].getAttribute("href")?.replace(origin.value, ""))
  }  
}
</script>

<template>
  <div class="VPNavBarSearch" >
    <!-- <SearchBox /> -->
    <ClientOnly>
      <Teleport to="body">
        <div v-show="open" class="VPPluginSearch-modal-back" @click="open = false" @keydown="handleNavigation">
          <div class="VPPluginSearch-modal" @click.stop ref="modal" >
            <form class="DocSearch-Form">
              <label
                class="DocSearch-MagnifierLabel"
                for="docsearch-input"
                id="docsearch-label"
                ><svg
                  width="20"
                  height="20"
                  class="DocSearch-Search-Icon"
                  viewBox="0 0 20 20"
                >
                  <path
                    d="M14.386 14.386l4.0877 4.0877-4.0877-4.0877c-2.9418 2.9419-7.7115 2.9419-10.6533 0-2.9419-2.9418-2.9419-7.7115 0-10.6533 2.9418-2.9419 7.7115-2.9419 10.6533 0 2.9419 2.9418 2.9419 7.7115 0 10.6533z"
                    stroke="currentColor"
                    fill="none"
                    fill-rule="evenodd"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                  ></path>
                </svg>
              </label>

              <input
                class="DocSearch-Input"
                aria-autocomplete="both"
                aria-labelledby="docsearch-label"
                id="docsearch-input"
                autocomplete="off"
                autocorrect="off"
                autocapitalize="off"
                enterkeyhint="search"
                spellcheck="false"
                autofocus="true"
                v-model="searchTerm"
                :placeholder=placeholder
                maxlength="64"
                type="search"
                ref="input"
              />
            </form>
            <div class="VPPluginSearch-search-list">
              <div class="search-group"
                v-for="(group, groupKey) of groupedResults"
                :key="groupKey"                
              >
                <span class="VPPluginSearch-search-group">{{
                  groupKey
                    ? groupKey.toString()[0].toUpperCase() +
                      groupKey.toString().slice(1) 
                    : "Home"
                }}</span>
                <a
                  :href="origin + item.link"
                  v-for="(item, index) in group"
                  :key="item.id"
                  @click="cleanSearch"                  
                  @mouseenter="focused = item.id"
                  :class="{'link-focused':focused == item.id}"
                  :id="index.toString()"
                >
                  <div class="VPPluginSearch-search-item">
                    <span class="VPPluginSearch-search-item-icon">{{
                      item.link.includes("#") ? "#" : "▤"
                    }}</span>
                    <div style="width: 100%">
                      <h3>{{ item.title }}</h3>
                      <div class="VPPluginSearch-search-item-preview"><div v-html="item.preview"></div></div>
                    </div>
                    <span class="VPPluginSearch-search-item-icon">↪</span>
                  </div>
                </a>
              </div>
            </div>
            <img class="VPPluginSearch-flex-logo" src="./flex-logo.svg" alt="flex logo"/>
          </div>
        </div>
      </Teleport>
    </ClientOnly>
    <div id="docsearch" @click="openSearch()">
      <button
        type="button"
        class="DocSearch DocSearch-Button"
        aria-label="Search"
      >
        <span class="DocSearch-Button-Container">
          <svg
            width="20"
            height="20"
            class="DocSearch-Search-Icon"
            viewBox="0 0 20 20"
          >
            <path
              d="M14.386 14.386l4.0877 4.0877-4.0877-4.0877c-2.9418 2.9419-7.7115 2.9419-10.6533 0-2.9419-2.9418-2.9419-7.7115 0-10.6533 2.9418-2.9419 7.7115-2.9419 10.6533 0 2.9419 2.9418 2.9419 7.7115 0 10.6533z"
              stroke="currentColor"
              fill="none"
              fill-rule="evenodd"
              stroke-linecap="round"
              stroke-linejoin="round"
            ></path>
          </svg>
          <span class="DocSearch-Button-Placeholder">{{buttonLabel}}</span>
        </span>
        <span class="DocSearch-Button-Keys">
          <span class="DocSearch-Button-Key" ref="metaKey">Meta</span>
          <span class="DocSearch-Button-Key">K</span>
        </span>
      </button>
    </div>
  </div>
</template>

<style>
.VPPluginSearch-flex-logo{
  width: 80px;
  margin-left: calc(100% - 90px);
  padding-bottom: 10px;
}

.VPPluginSearch-search-list {
  padding: 1rem;
  max-height: calc(100vh - 230px);
  overflow-x: auto;
}

.VPPluginSearch-search-item-icon {
  font-family: none;
  align-self: center;
  padding: 0 1rem 0 0;
  font-size: x-large;
}

.VPPluginSearch-search-list > div {
  color: var(--vp-c-brand-dark);
  font-weight: bold;
}

.VPPluginSearch-search-item {
  padding: 0.25rem 1rem;
  margin: 8px 0 0 0;
  border: solid 1px;
  border-radius: 6px;
  display: flex;

  border-color: var(--vp-custom-block-details-border);
  color: var(--vp-custom-block-details-text);
  background-color: var(--vp-custom-block-details-bg);
}

.VPPluginSearch-search-item .VPPluginSearch-search-item-preview {
  margin: 0px;
  font-size: smaller;
  color: var(--c-text-light-3);
}

a.link-focused .VPPluginSearch-search-item,
.VPPluginSearch-search-item:hover {
  color: #fff;
  background: var(--vp-local-search-highlight-bg);
}

a.link-focused .VPPluginSearch-search-item,
.VPPluginSearch-search-item:hover > .VPPluginSearch-search-item-preview {
  color: #fff;
}

/* .dark .search-item > .VPPluginSearch-search-item-preview {
  color: var(--c-text-light-2);
} */

.VPNavBarSearch {
  display: flex;
  align-items: center;
  padding-left: 16px;
}

.DocSearch-MagnifierLabel {
  margin: 16px;
  color: var(--c-brand-light);
  stroke-width: 2px;
}

.DocSearch-Input {
  appearance: none;
  background: #58565636;
  border: solid 1px var(--c-brand-light);
  color: var(--docsearch-text-color);
  flex: 1;
  font: inherit;
  font-size: 1.2em;
  height: 100%;
  outline: none;
  padding: 0 0 0 8px;
  width: 80%;
  margin: 8px;
  padding: 8px;
  border-radius: 6px;
}

.dark .DocSearch-Input {
  color: var(--vp-c-text-1);
  /* background-color: var(--vp-c-bg); */
}

.VPPluginSearch-modal-back {
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
  background: #545454b3;
  position: fixed;
  z-index: 65;
}

.dark .VPPluginSearch-modal {
  background-color: #242424;
}

.VPPluginSearch-modal {
  background-color: #f9f9f9;
  border-radius: 6px;
  box-shadow: none;
  flex-direction: column;
  margin: 80px auto auto;
  max-width: 560px;
  position: relative;
  /* box-shadow: inset 1px 1px 0 0 hsla(0, 0%, 100%, 0.5), 0 3px 8px 0 #555a64; */
}

.DocSearch-Button-Keys {
  display: flex;
  min-width: calc(40px + 0.8em);
}

@media (min-width: 768px) {
  .DocSearch-Button .DocSearch-Button-Key {
    display: inline-block;
  }
}

@media (max-width: 768px) {
  .VPPluginSearch-modal {
    max-width: 100%;
    border-radius: 0px;
  }
}

.dark .DocSearch-Form {
  background-color: var(--vt-c-bg-mute);
}
.DocSearch-Form {
  background-color: #fff;
  border: 1px solid var(--vt-c-brand);
  padding: 6px;
}
.DocSearch-Form {
  align-items: center;
  background: var(--docsearch-searchbox-focus-background);
  border-radius: 4px;
  box-shadow: var(--docsearch-searchbox-shadow);
  display: flex;
  height: var(--docsearch-searchbox-height);
  margin: 0;
  padding: 0 var(--docsearch-spacing);
  position: relative;
  width: 100%;
}
.DocSearch-Container,
.DocSearch-Container * {
  box-sizing: border-box;
}

.DocSearch-Button .DocSearch-Button-Key {
  margin-top: 2px;
  border: 1px solid var(--vt-c-divider);
  border-right: none;
  border-radius: 4px 0 0 4px;
  display: none;
  padding-left: 6px;
  height: 22px;
  line-height: 22px;
  transition: color 0.5s, border-color 0.5s;
  min-width: 0;
}
.DocSearch-Button-Key {
  font-size: 12px;
  font-weight: 500;
  height: 20px;
  margin: 0;
  width: auto;
  color: var(--vt-c-text-3);
  transition: color 0.5s;
  display: inline-block;
  padding: 0 1px;
}
.DocSearch-Button-Key {
  align-items: center;
  background: var(--c-brand-light);
  border-radius: 3px;
  box-shadow: var(--c-brand);
  color: var(--docsearch-muted-color);
  display: flex;
  height: 18px;
  justify-content: center;
  margin-right: 0.4em;
  padding-bottom: 2px;
  position: relative;
  top: -1px;
  width: 20px;
}

body.dark .DocSearch-Button:hover {
  box-shadow: none;
}

.DocSearch {
  --docsearch-primary-color: var(--vt-c-brand);
  --docsearch-highlight-color: var(--docsearch-primary-color);
  --docsearch-text-color: var(--vt-c-text-1);
  --docsearch-muted-color: #ebebeb99;
  --docsearch-searchbox-shadow: none;
  --docsearch-searchbox-focus-background: transparent;
  --docsearch-key-gradient: transparent;
  --docsearch-key-shadow: none;
  --docsearch-modal-background: var(--vt-c-bg-soft);
  --docsearch-footer-background: var(--vt-c-bg);
}

.dark .DocSearch {
  --docsearch-modal-shadow: none;
  --docsearch-footer-shadow: none;
  --docsearch-logo-color: #ebebeb99;
  --docsearch-hit-background: #2f2f2f;
  --docsearch-hit-color: #ebebeb99;
  --docsearch-hit-shadow: none;
}

.dark .DocSearch-Footer {
  border-top: 1px solid var(--vt-c-divider);
}

.DocSearch-Form {
  background-color: white;
  border: 1px solid var(--vt-c-brand);
}

.DocSearch-Button-Container {
  align-items: center;
  display: flex;
}

.DocSearch-Button {
  display: flex;
  justify-content: center;
  align-items: center;
  margin: 0;
  width: 48px;
  height: 55px;
  background: transparent;
  border: none;
}

.DocSearch-Button:hover {
  background: transparent;
}
.DocSearch-Button:focus {
  outline: 1px dotted;
  outline: 5px auto -webkit-focus-ring-color;
}
.DocSearch-Button:focus:not(:focus-visible) {
  outline: none !important;
}

@media (min-width: 768px) {
  .DocSearch-Button {
    justify-content: flex-start;
    width: 100%;
  }
}

.DocSearch-MagnifierLabel {
  color: var(--vp-c-brand-dark);
}

.DocSearch-Button .DocSearch-Search-Icon {
  transition: color 0.5s;
  fill: currentColor;
  width: 18px;
  height: 18px;
  position: relative;
}

@media (min-width: 768px) {
  .DocSearch-Button .DocSearch-Search-Icon {
    top: 1px;
    margin-right: 10px;
    width: 15px;
    height: 15px;
  }
}

.DocSearch-Button:hover .DocSearch-Search-Icon {
  color: var(--vt-c-text-1);
}

.DocSearch-Button-Placeholder {
  transition: color 0.5s;
  font-size: 13px;
  font-weight: 500;
  color: #66666699;
  display: none;
  padding: 0 10px 0 0;
}

@media (min-width: 960px) {
  .DocSearch-Button-Placeholder {
    display: inline-block;
  }
}

.DocSearch-Button:hover .DocSearch-Button-Placeholder {
  color: var(--vt-c-text-1);
}

.DocSearch-Button .DocSearch-Button-Key {
  margin-top: 2px;
  border: 1px solid var(--vt-c-divider);
  border-right: none;
  border-radius: 4px 0 0 4px;
  display: none;
  padding-left: 6px;
  height: 22px;
  line-height: 22px;
  transition: color 0.5s, border-color 0.5s;
  min-width: 0;
}

.DocSearch-Button .DocSearch-Button-Key + .DocSearch-Button-Key {
  border-right: 1px solid var(--vt-c-divider);
  border-left: none;
  border-radius: 0 4px 4px 0;
  padding-left: 2px;
  padding-right: 6px;
}

.DocSearch-Button:hover .DocSearch-Button-Key {
  /* border-color: var(--vt-c-brand-light); */
  color: var(--vt-c-brand-light);
}

@media (min-width: 768px) {
  .DocSearch-Button .DocSearch-Button-Key {
    display: inline-block;
  }
}

.DocSearch-Button-Key {
  font-size: 12px;
  font-weight: 500;
  height: 20px;
  margin: 0;
  width: auto;
  color: var(--vt-c-text-3);
  transition: color 0.5s;
  display: inline-block;
  padding: 0 1px;
}

.VPPluginSearch-search-group{
  color: var(--vp-c-brand-1)
}
</style>
