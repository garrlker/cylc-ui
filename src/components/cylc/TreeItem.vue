<template>
  <div>
    <v-row
        v-show="depth >= minDepth"
        :class="getNodeClass()"
        :style="getNodeStyle()"
    >
      <!-- the node's left icon; used for expand/collapse -->
      <v-flex
        class="node-expand-collapse-button"
        shrink
        v-if="hasChildren"
        @click="typeClicked"
        :style="getTypeStyle()"
      >{{ isExpanded ? '&#9661;' : '&#9655;' }}</v-flex>
      <!-- the node value -->
      <!-- TODO: revisit these values that can be replaced by constants later (and in other components too). -->
      <v-layout class="node-data" @click="nodeClicked" row wrap v-if="node.__type === 'cyclepoint'">
        <v-flex shrink>
          <task :status="node.state" :progress=0 />
        </v-flex>
        <v-flex grow>
          <span class="mx-1">{{ node.name }}</span>
        </v-flex>
      </v-layout>
      <v-layout class="node-data" @click="nodeClicked" row wrap v-else-if="node.__type === 'task'">
        <v-flex shrink>
          <task :status="node.state" :progress="node.progress" />
        </v-flex>
        <v-flex shrink>
          <span class="mx-1">{{ node.name }}</span>
        </v-flex>
        <v-flex grow ml-4 v-if="!isExpanded">
          <!-- Task summary -->
          <job
              v-for="(task, index) in node.children"
              :key="`${task.id}-summary-${index}`"
              :status="task.state" />
        </v-flex>
      </v-layout>
      <v-layout class="node-data" layout column v-else-if="node.__type === 'job'">
        <v-layout @click="jobNodeClicked" row wrap>
          <v-flex shrink>
            <job :status="node.state" />
          </v-flex>
          <v-flex xs2 md1 lg1>
            <span class="mx-1">{{ node.name }}</span>
          </v-flex>
          <v-flex grow>
            <span class="grey--text">{{ node.host }}</span>
          </v-flex>
        </v-layout>
        <!-- leaf node -->
      </v-layout>
      <v-layout class="node-data" row wrap v-else>
        <span @click="nodeClicked" class="mx-1">{{ node.name }}</span>
      </v-layout>
    </v-row>
    <v-container class="leaf" v-if="displayLeaf && node.__type === 'job'">
      <div class="arrow-up" :style="getLeafTriangleStyle()"></div>
      <v-col class="leaf-data font-weight-light py-4">
        <v-row v-for="leafProperty in leafProperties" :key="leafProperty.id" no-gutters>
          <v-col xs="4" sm="4" md="4" lg="2" xl="2" no-wrap>
            <span class="px-4">{{ leafProperty.title }}</span>
          </v-col>
          <v-col wrap>
            <span class="grey--text">{{ node[leafProperty.property] }}</span>
          </v-col>
        </v-row>
      </v-col>
    </v-container>
    <span v-show="isExpanded">
      <!-- component recursion -->
      <TreeItem
          v-for="child in node.children"
          ref="treeitem"
          :key="child.id"
          :node="child"
          :depth="depth + 1"
          :hoverable="hoverable"
          :min-depth="minDepth"
          :initialExpanded="initialExpanded"
          v-on:tree-item-created="$emit('tree-item-created', $event)"
          v-on:tree-item-expanded="$emit('tree-item-expanded', $event)"
          v-on:tree-item-collapsed="$emit('tree-item-collapsed', $event)"
          v-on:tree-item-clicked="$emit('tree-item-clicked', $event)"
      ></TreeItem>
    </span>
  </div>
</template>

<script>
import Task from '@/components/cylc/Task'
import Job from '@/components/cylc/Job'

/**
 * Offset used to move nodes to the right or left, to represent the nodes hierarchy.
 * @type {number} integer
 */
const NODE_DEPTH_OFFSET = 30

export default {
  name: 'TreeItem',
  components: {
    task: Task,
    job: Job
  },
  props: {
    node: {
      type: Object,
      required: true
    },
    depth: {
      type: Number,
      default: 0
    },
    minDepth: {
      type: Number,
      default: 0
    },
    hoverable: Boolean,
    initialExpanded: {
      type: Boolean,
      default: true
    }
  },
  data () {
    return {
      active: false,
      selected: false,
      isExpanded: this.initialExpanded,
      leafProperties: [
        {
          title: 'host id',
          property: 'host'
        },
        {
          title: 'job id',
          property: 'batchSysJobId'
        },
        {
          title: 'batch sys',
          property: 'batchSysName'
        },
        {
          title: 'submit time',
          property: 'submittedTime'
        },
        {
          title: 'start time',
          property: 'startedTime'
        },
        {
          title: 'finish time',
          property: 'finishedTime'
        },
        {
          title: 'latest message',
          property: 'latestMessage'
        }
      ],
      displayLeaf: false
    }
  },
  computed: {
    hasChildren () {
      return Object.prototype.hasOwnProperty.call(this.node, 'children')
    }
  },
  created () {
    this.$emit('tree-item-created', this)
  },
  beforeDestroy () {
    if (Object.prototype.hasOwnProperty.call(this.$refs, 'treeitem')) {
      this.$refs.treeitem.map((treeitem) => {
        treeitem.destroy()
      })
    }
  },
  beforeMount () {
    if (Object.prototype.hasOwnProperty.call(this.node, 'expanded')) {
      this.isExpanded = this.node.expand
      this.emitExpandCollapseEvent(this.isExpanded)
    }
  },
  methods: {
    destroy () {
      // $destroy will trigger beforeDestroy and destroyed
      this.$destroy(/* destroy from DOM as well */ true)
    },
    typeClicked () {
      this.isExpanded = !this.isExpanded
      this.emitExpandCollapseEvent(this.isExpanded)
    },
    /**
     * Emits an event `tree-item-expanded` if `expanded` is true, or emits
     * `tree-item-collapsed` if `expanded` is false.
     * @param {boolean} expanded whether the node is expanded or not
     */
    emitExpandCollapseEvent (expanded) {
      if (expanded) {
        this.$emit('tree-item-expanded', this)
      } else {
        this.$emit('tree-item-collapsed', this)
      }
    },
    getTypeStyle () {
      const styles = {}
      if (this.hasChildren) {
        styles.cursor = 'pointer'
      }
      return styles
    },
    /**
     * Handler for when any node of the tree was clicked, except jobs.
     * @param {event} e event
     */
    nodeClicked (e) {
      this.$emit('tree-item-clicked', this)
    },
    /**
     * Handler for when a job node was clicked.
     * @param {event} e event
     */
    jobNodeClicked (e) {
      this.displayLeaf = !this.displayLeaf
    },
    getNodeStyle () {
      // we need to compensate for the minimum depth set by the user, subtracting it
      // from the node depth.
      const depthDifference = this.depth - this.minDepth
      return {
        'padding-left': `${depthDifference * NODE_DEPTH_OFFSET}px`
      }
    },
    /**
     * All nodes have the same padding-left. However, the job leaf node needs special care, as it will occupy the
     * whole content area.
     *
     * For this, we calculate it similarly to `getNodeStyle` but doing the reverse, to move the element to the
     * left, instead of moving it to the right. Using `depth` to calculate the exact location for the element.
     */
    getLeafTriangleStyle () {
      const depthDifference = this.depth - this.minDepth
      return {
        'margin-left': `${depthDifference * NODE_DEPTH_OFFSET}px`
      }
    },
    getNodeClass () {
      return {
        node: true,
        'node--hoverable': this.hoverable,
        'node--active': this.active,
        'ml-3': true
      }
    }
  }
}
</script>

<style scoped lang="scss">
@import '~vuetify/src/styles/styles.sass';

$active-color: #BDD5F7;

@mixin active-state() {
  background-color: $active-color;
  &:hover {
    background-color: $active-color;
  }
}

@mixin states() {
  &:hover {
    background-color: map-get($grey, 'lighten-3');
  }
}

.node {
  line-height: 1.8em;

  &--hoverable {
    @include states()
  }

  &--active {
    @include active-state()
  }

  .node-data {
    margin-left: 6px;
  }
}
.type {
  margin-right: 10px;
}

$arrow-size: 15px;
$leaf-background-color: map-get($grey, 'lighten-3');

.leaf {
  padding: 0;
  margin: 0;
  .arrow-up {
    width: 0;
    height: 0;
    border-left: $arrow-size solid transparent;
    border-right: $arrow-size solid transparent;
    border-bottom: $arrow-size solid $leaf-background-color;
    display: flex;
    flex-wrap: nowrap;
  }
  .leaf-data {
    background-color: $leaf-background-color;
  }
}
</style>
