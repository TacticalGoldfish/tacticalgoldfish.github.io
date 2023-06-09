<template>
  <!--spans and anchors-->
  <span v-if="element.type == 'span'" :style="element.style ? element.style : ''" :class="element.class ? element.class : ''">{{ element.text }}</span>
  <a v-else-if="element.type == 'anchor' && element.targetModule" :style="element.style ? element.style : ''" :class="element.class ? element.class : ''" @click.prevent="openDetails(element.targetModule, element.targetId)">{{ element.text }}</a>
  <a v-else-if="element.type == 'anchor' && element.href" :style="element.style ? element.style : ''" :class="element.class ? element.class : ''" :href="element.href" target="_blank" >{{ element.text }}</a>
  
  <!--row layout-->
  <v-row v-else-if="element.type == 'row'" :style="element.style ? element.style : ''" :class="element.class ? element.class : ''">
    {{ element.text ? element.text : '' }}
    <DynamicElement 
      v-for="child, chi in element.children" 
      :element="child" 
      :layer="layer + 1" 
      :key="`${child.type}-element-${chi}-layer-${layer}`" 
      @openDetails="(moduleShortcut, id) => openDetails(moduleShortcut, id)"
       />
  </v-row>
  <!--column layout-->
  <v-col v-else-if="element.type == 'column'" :lg="element.colslg" :md="element.colsmd" :sm="element.colssm" :xs="element.colsxs" :cols="element.cols" :style="element.style ? element.style : ''" :class="element.class ? element.class : ''" >
    {{ element.text ? element.text : '' }}
    <DynamicElement 
      v-for="child, chi in element.children" 
      :element="child" 
      :layer="layer + 1" 
      :key="`${child.type}-element-${chi}-layer-${layer}`"  
      @openDetails="(moduleShortcut, id) => openDetails(moduleShortcut, id)"/>
  </v-col>
  <br v-else-if="element.type == 'newLine'" />

  <!--basic image-->
  <v-img v-else-if="element.type == 'image'" :src="`./img/${element.source}`" :height="element.height" :contain="element.contain ? element.contain : false" />

  
  <!--map - will only show if grid details are set-->
  <div v-else-if="element.type == 'map' && element.xyGrid && element.xyGrid.length == 2" v-bind:style="{ height: '100%', width: '100%' }" style="justify-content: center; display: flex;" ref="container">
    <Konva-stage ref="stage" :config="stageSize">
      <Konva-layer ref="gridLayer">
        <Konva-image :config="imageConfig" :src="`./img/${element.source}`" :height="element.height" :contain="element.contain ? element.contain : false"/> 
        <!--set up the grid-->
        <template>
          <Konva-line
            v-for="n in (element.xyGrid[0] + 1)"
            :key="`element-${element.type}-layer-${layer}-vert-${n}`"
              :config="{
                points: [(stageSize.width / element.xyGrid[0] * n), 0, (stageSize.width / element.xyGrid[0] * n), stageSize.height],
                strokeWidth: 2,
                stroke: `${element.xyGridColor}`,
              }"
            />
          <Konva-line
            v-for="n in (element.xyGrid[1] + 1)"
            :key="`element-${element.type}-layer-${layer}-horiz-${n}`"
            :config="{
              points: [0, (stageSize.height / element.xyGrid[1] * n), stageSize.width, (stageSize.height / element.xyGrid[1] * n)],
              strokeWidth: 2,
              stroke: `${element.xyGridColor}`,
            }"
          />
        </template>
        <!--set up pins if necessary-->
        <template v-if="element.pins">
          <Konva-label 
            v-for="pin, n in element.pins"
            :key="`element-${element.type}-layer-${layer}-pin-${n}`"
            @pointerclick="openDetails(pin.targetModule, pin.targetId)"
            @mouseenter="handleMouseEnter"
            @mouseleave="handleMouseLeave"
            :config="{
              x: (stageSize.width / element.xyGrid[0] * pin.x),
              y: (stageSize.height / element.xyGrid[1] * pin.y),
              opacity: 0.75
            }"
          >
            <Konva-tag 
              :config="{
                fill: 'black',
                pointerDirection: 'down',
                pointerWidth: 10,
                pointerHeight: 10,
                lineJoin: 'round',
                shadowColor: 'black',
                shadowBlur: 10,
                shadowOffset: 10,
                shadowOpacity: 0.5,
              }" 
            />
            <Konva-text 
              :config="{
              text: pin.text,
              fontFamily: 'Calibri',
              fontSize: 18,
              padding: 5,
              fill: 'white',
            }" />
          </Konva-label>
        </template>
        <!--set up polygons necessary-->
        <template v-if="element.polygons">
          <Konva-line
            v-for="polygon, index in element.polygons"
            :key="`element-${element.type}-layer-${layer}-polygon-${index}`"
            :config="{
              points: calculatePolygonPoints(polygon),
              strokeWidth: 1,
              stroke: `${polygon.strokeColor}`,
              closed: true,
              x: 0,
              y: 0,
              fill: polygon.fillColor
            }"
          />
          
        </template>
      </Konva-layer>
    </Konva-stage>
  </div>
</template>

<script>
  const width = window.innerWidth;
  const height = window.innerHeight;
  //let vm = {};
  import { unpackDescription } from '@/helpers/contentHelper';
  import DynamicElement from '@/components/DynamicElement';
  export default {
    name: 'DynamicElement',
    components: {
      DynamicElement
    },
    data: () => ({
      selectedCharacter: null,
      stageSize: {
        width: width,
        height: height
      },
      image: null,
    }),
    computed: {
      imageConfig() {
        return { image: this.image }
      }
    },
    props: {
      element: {
        type: Object,
        default: () => {},
      },
      layer: {
        type: Number,
        default: 0,
      }
    },
    created: function() {
      window.addEventListener("resize", this.changeRect);
    },
    watch: {
      element: {
        immediate: true,
        handler(val){
          if (!val) return;
          if (val.type == "map") {
            this.changeRect();
          }
        },
      }
    },
    methods: {
      unpackDescription(description){
        const descriptionElements = unpackDescription(description);
        return descriptionElements;
      },
      openDetails(module, id){
        this.$emit('openDetails', module, id);
      },
      async changeRect() {
        await this.$nextTick();
        const container = this.$refs.container;

        if (!container) {
          return;
        }
        const image = new window.Image();
        image.src = `./img/${this.element.source}`;
        image.onload = () => {
          const aspectRatio = container.offsetParent.offsetWidth / image.width;

          const height = image.height * aspectRatio;
          const width = container.offsetWidth;
          image.width = width;
          image.height = height;

          this.stageSize.width = width;
          this.stageSize.height = height;
          this.image = image;
          
        };
      },
      handleMouseEnter(e) {
        const stage = e.target.getStage();
        stage.container().style.cursor = "pointer";
      },
      handleMouseLeave(e) {
        const stage = e.target.getStage();
        stage.container().style.cursor = "default";
      },
      calculatePolygonPoints(polygon) {
        //[0, (stageSize.height / element.xyGrid[1] * n), stageSize.width, (stageSize.height / element.xyGrid[1] * n)]
        const width = this.stageSize.width
        const height = this.stageSize.height
        const tileWidth = width / this.element.xyGrid[0]
        const tileHeight = height / this.element.xyGrid[1]
        let coordinates = [];
        polygon.points.forEach((coordinate, index) => {
          if(index % 2) coordinates.push(coordinate * tileHeight); //Y coordinate
          else coordinates.push(coordinate * tileWidth); //Y coordinate
        });
        return coordinates;
      }
    }
  }
</script>
<style>
  .canvas-overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    pointer-events: none;
    width: 100%;
    height: 100%
  }

  .canvas-wrapper {
    position: relative;
  }
</style>