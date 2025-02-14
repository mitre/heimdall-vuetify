<template>
  <v-container fluid>
    <v-row dense>
      <v-col :cols="4">
        NIST SP 800-53 Security Control Coverage
      </v-col>
      <v-col :cols="8">
        <v-btn @click="up" :disabled="!allow_up" block x-small>
          <v-icon v-if="allow_up"> mdi-arrow-left </v-icon>
          {{ 'NIST-SP-800-53 -> ' + value.join(' -> ') }}
        </v-btn>
      </v-col>
    </v-row>
    <v-row>
      <v-col :cols="12" v-resize:debounce="on_resize">
        <svg id="chartBody" :width="width" :height="height">
          <g
            style="shape-rendering: crispEdges;"
            preserveAspectRatio="xMidYMid meet"
          >
            <!-- The body -->
            <Cell
              :selected_control_id="selected_control"
              :node="selected_node"
              :scales="scales"
              :depth="0"
              @select-node="select_node"
            />
          </g>
        </svg>
      </v-col>
    </v-row>
  </v-container>
</template>

<script lang="ts">
//               preserveAspectRatio="xMidYMid meet"
import Vue from 'vue';
import Component from 'vue-class-component';
import {ControlStatus, HDFControl, nist} from 'inspecjs';
import * as d3 from 'd3';
import {FilteredDataModule, TreeMapState} from '@/store/data_filters';
import {
  TreemapNode,
  build_nist_tree_map,
  is_leaf,
  is_parent
} from '@/utilities/treemap_util';
import {HierarchyRectangularNode, tree} from 'd3';
import Cell, {XYScale} from '@/components/cards/treemap/Cell.vue';
//@ts-ignore
import resize from 'vue-resize-directive';
import {ColorHackModule} from '../../../store/color_hack';
import {compare_arrays} from '@/utilities/helper_util';

// We declare the props separately to make props types inferable.
const TreemapProps = Vue.extend({
  props: {
    value: {
      type: Array, // Of type Array<TreeMapNode>, representing current descent path
      required: true
    },
    /*
    path: {
      type: Array, // Of type TreeMapState, representing current descent path
      required: true
    },
    */
    selected_control: {
      // Represents control id
      required: false
    },
    filter: {
      type: Object, // Of type Filter
      required: true
    }
  }
});

// Respects a v-model of type TreeMapState
@Component({
  components: {
    Cell
  },
  directives: {
    resize
  }
})
export default class Treemap extends TreemapProps {
  /** The svg internal coordinate space */
  width: number = 1600;
  height: number = 530;

  /** Typed getter on current spec path */
  get _state(): TreeMapState {
    return this.value as TreeMapState;
  }

  /** The currently selected treemap node. Wrapped to avoid initialization woes */
  get selected_node(): d3.HierarchyRectangularNode<TreemapNode> {
    // Get typed versions of the curr state
    // Set curr to root
    let curr = this.treemap_layout;
    let depth = 0;

    try {
      for (; depth < this._state.length; depth++) {
        // If the current has no children, then just bail here
        if (curr.children === undefined) {
          throw 'no children to go into';
        }

        // Fetch the next path spec
        let next_specifiers = this._state.slice(0, depth + 1);

        let new_curr = curr.children.find(child => {
          if (is_parent(child.data)) {
            let ss_a = child.data.nist_control.sub_specifiers;
            return (
              compare_arrays(ss_a, next_specifiers, (a, b) =>
                a.localeCompare(b)
              ) === 0
            );
          } else {
            return false; // We cannot go into a leaf (OR CAN WE? MUST DECIDE, AT SOME POINT)
          }
        });
        if (new_curr) {
          if (new_curr.children && new_curr.children.length) {
            curr = new_curr;
          } else {
            throw 'empty';
          }
        } else {
          throw 'truncate';
        }
      }
    } catch (some_traversal_error) {
      // Slice to last successful depth. Slice is non inclusive so this works
      this.set_path(this._state.slice(0, depth));
    }

    // Return as deep as we travelled
    return curr;
  }

  /** Get our viewbox */
  get view_box(): string {
    return `0 0 ${this.width} ${this.height}`;
  }

  /** Get our scales */
  get scales(): XYScale {
    return {
      scale_x: d3
        .scaleLinear()
        .domain([this.selected_node.x0, this.selected_node.x1])
        .range([0, this.width]),
      scale_y: d3
        .scaleLinear()
        .domain([this.selected_node.y0, this.selected_node.y1])
        .range([0, this.height])
    };
  }

  /** Generates a d3 heirarchy structure, with appropriate bounds to our width
   *  detailing all of the controls in the nist hash */
  get treemap_layout(): d3.HierarchyRectangularNode<TreemapNode> {
    // Get the currejnt filtered data
    let controls = FilteredDataModule.controls(this.filter);

    // Build the map
    let hierarchy = build_nist_tree_map(controls, ColorHackModule);
    let treemap = d3
      .treemap<TreemapNode>()
      .size([this.width, this.height])
      .round(false)
      .paddingInner(0)(hierarchy);
    return treemap;
  }

  // Callbacks for our tree
  select_node(n: d3.HierarchyRectangularNode<TreemapNode>): void {
    // If it is a leaf, then select it
    let new_state = [...this._state];
    if (is_leaf(n.data)) {
      let id = n.data.control.data.id;
      if (id !== this.selected_control) {
        this.$emit('update:selected_control', id);
      } else {
        this.$emit('update:selected_control', null);
      }
    } else {
      // Otherwise, dive away. Set course for the leading title
      let cntrl = n.data.nist_control;
      if (cntrl) {
        this.set_path(cntrl.sub_specifiers);
      }
    }
  }

  /** Submits an event to go up one node */
  up(): void {
    if (this._state.length) {
      // Slice and dice, baybee
      this.set_path(this._state.slice(0, this._state.length - 1));

      // Also clear selected
      this.$emit('update:selected_control', null);
    }
  }

  /** Typed method to wrap changes in the depth */
  set_path(path_spec: TreeMapState) {
    this.$emit('input', path_spec);
  }

  /** Controls whether we should allow up */
  get allow_up(): boolean {
    return this._state.length > 0;
  }

  /** Called on resize */
  on_resize(elt: any) {
    if (elt.clientWidth !== undefined && elt.clientWidth > 1) {
      this.width = elt.clientWidth - 24;
    }
  }
}
</script>

<style scoped>
text {
  pointer-events: none;
  font-weight: bold;
  font-size: 1.1em;
  fill: 'primary';
}

rect {
  fill: none;
}
</style>
