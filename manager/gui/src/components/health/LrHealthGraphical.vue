<!--
* CORTX-Halo: Halo Management GUI.
* Copyright (c) 2022 Seagate Technology LLC and/or its Affiliates
* This program is free software: you can redistribute it and/or modify
* it under the terms of the GNU Affero General Public License as published
* by the Free Software Foundation, either version 3 of the License, or
* (at your option) any later version.
* This program is distributed in the hope that it will be useful,
* but WITHOUT ANY WARRANTY; without even the implied warranty of
* MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
* GNU Affero General Public License for more details.
* You should have received a copy of the GNU Affero General Public License
* along with this program. If not, see <https://www.gnu.org/licenses/>.
* For any questions about this software or licensing,
* please email opensource@seagate.com.
-->
<template>
  <div class="graphical-view-container">
    <div class="export-feature" v-if="showExport">
      <SgtDropdown
        placeholder="EXPORT AS"
        :dropdownOptions="exportOptions"
        v-model="selectedExport"
      />
      <v-btn
        color="csmprimary"
        class="white--text export-btn"
        @click="handleExport"
        :disabled="!selectedExport"
        >Export</v-btn
      >
    </div>

    <div id="health_tree_container" :style="healthTreeContainerDim"></div>
  </div>
</template>
<script lang="ts">
import { Component, Mixins } from "vue-property-decorator";
import ClusterManagementMixin from "../../mixins/ClusterManagement";
import * as d3 from "d3";
import HealthCardBuilder from "./LrHealthCardBuilder";
import {
  downloadSVGAsText,
  downloadSVGAsPNG,
  downloadSVGAsJPEG,
  downloadSVGAsPDF,
} from "../../utils/SVGExport";
import { Dimensions } from "@/utils/LrUtilFunctions";
import SgtDropdown from "@/lib/components/SgtDropdown/SgtDropdown.vue";
@Component({
  name: "LrHealthGraphical",
  components: { SgtDropdown },
})
export default class LrHealthGraphical extends Mixins(ClusterManagementMixin) {
  public healthTreeContainerDim: any = {
    height: "0px",
    width: "0px",
    overflow: "auto",
  };
  public healthTreePathLine: any;
  public healthTreePathLineCoordinates: any[] = [];
  public resourceCardDim: any = {
    width: 316,
    height: 120,
    headerHeight: 50,
  };
  public treeContainerDim: any = {
    width: 0,
    height: 20,
    paddingX: 20,
    paddingY: 20,
  };
  public resource_health: any;
  public outerG: any;

  public exportOptions = ["PDF", "PNG", "JPEG", "SVG"];
  public selectedExport = "";

  public async mounted() {
    this.calculateDimensions();
    this.resource_health = await this.getHealthData(true);

    this.healthTreePathLine = d3.line().defined((d) => d !== null);
    const nodes: any[] = this.traverse();
    this.initSVG();
    this.buildSVG(nodes);
  }

  get showExport() {
    return this.$route.path === "/health";
  }

  private calculateDimensions() {
    const mainContentDim: any = Dimensions.getInstance().getContentDimension();
    this.healthTreeContainerDim.height = `${mainContentDim.height - 69}px`;
    this.healthTreeContainerDim.width = `${mainContentDim.width - 110}px`;
  }

  private initSVG() {
    const svgWidth =
      this.treeContainerDim.width + this.resourceCardDim.width + 5;
    const svgHeight = this.treeContainerDim.height + 70;

    const healthTreeContainerSVG = d3
      .select("#health_tree_container")
      .append("svg")
      .attr("width", svgWidth)
      .attr("height", svgHeight)
      .attr("xmlns", "http://www.w3.org/2000/svg")
      .attr("xmlns:xlink", "http://www.w3.org/1999/xlink")
      .attr("class", "health-tree");
    this.outerG = healthTreeContainerSVG
      .append("g")
      .attr("id", "outer_g")
      .attr("width", svgWidth)
      .attr("height", svgHeight);

    // Set zoom
    const zoom: any = d3.zoom().on("zoom", (event: any) => {
      this.outerG.attr("transform", event.transform);
    });
    healthTreeContainerSVG.call(zoom);
    if (+this.healthTreeContainerDim.width.slice(0, -2) < 1200) {
      this.outerG.attr("transform", `translate(0,0) scale(0.7)`);
    } else if (+this.healthTreeContainerDim.width.slice(0, -2) < 1450) {
      this.outerG.attr("transform", `translate(0,0) scale(0.8)`);
    }
  }

  private buildSVG(nodes: any[]) {
    const enterJoin: any = (enter: any) => {
      const healthCardG = enter
        .append("g")
        .attr("class", "g_health_card")
        .attr("id", (d: any) => `g_${d.treeNodeId}`)
        .attr("transform", "translate(0,0)");
      HealthCardBuilder.build(
        healthCardG,
        this.expandCollapse,
        this.performAction
      );
      healthCardG
        .transition()
        .duration(750)
        .attr("transform", (data: any) => {
          return `translate(${data.x}, ${data.y})`;
        });
    };
    const updateJoin: any = (update: any) => {
      update.select(".expand_line").attr("stroke-width", (data: any) => {
        return data.sub_resources && data.sub_resources.length > 0 ? 0 : 2;
      });
      update
        .transition()
        .duration(750)
        .attr("transform", (data: any) => {
          return `translate(${data.x}, ${data.y})`;
        });
    };
    const exitJoin: any = (exit: any) => {
      exit.remove();
    };
    this.outerG
      .selectAll(".g_health_card")
      .data(nodes, (d: any) => d.treeNodeId)
      .join(enterJoin, updateJoin, exitJoin);

    this.outerG.select("#nodeConnectorPath").remove();
    this.outerG
      .append("path")
      .data([this.healthTreePathLineCoordinates])
      .transition()
      .delay(700)
      .attr("d", this.healthTreePathLine)
      .attr("id", "nodeConnectorPath")
      .style("fill", "none")
      .style("stroke", "#0832A0")
      .style("stroke-width", "2px")
      .style("stroke-dasharray", "5,5");
  }

  private traverse() {
    const resources = [];
    const stack = [];

    this.resource_health[0].x = this.treeContainerDim.paddingX;
    this.resource_health[0].y = this.treeContainerDim.paddingY;
    this.resource_health[0].parent_resource = null;
    this.resource_health[0].treeNodeId = "0";
    stack.push(this.resource_health[0]);

    let resource: any;
    while (stack.length > 0) {
      resource = stack.pop();

      if (resource.sub_resources && resource.sub_resources.length > 0) {
        resource.isLeafNode = false;
        resources.push(this.buildResourceObj(resource));
        for (let i = resource.sub_resources.length - 1; i >= 0; i--) {
          const sub_resource: any = resource.sub_resources[i];
          sub_resource.parent_resource = {
            x: resource.x,
            y: resource.y,
          };
          sub_resource.treeNodeId = `${resource.treeNodeId}${i}`;
          stack.push(sub_resource);
        }
      } else {
        resource.isLeafNode = true;
        resources.push(this.buildResourceObj(resource));
      }
    }

    return resources;
  }

  private buildResourceObj(resource: any) {
    resource.y = this.treeContainerDim.height;

    this.treeContainerDim.height = resource.isLeafNode
      ? this.treeContainerDim.height + this.resourceCardDim.height + 20
      : this.treeContainerDim.height;

    if (resource.parent_resource) {
      resource.x =
        resource.parent_resource.x + this.resourceCardDim.width + 100;
      if (resource.parent_resource.y === resource.y) {
        this.healthTreePathLineCoordinates.push([
          resource.x - 100,
          resource.y + this.resourceCardDim.headerHeight,
        ]);
        this.healthTreePathLineCoordinates.push([
          resource.x,
          resource.y + this.resourceCardDim.headerHeight,
        ]);
      } else {
        this.healthTreePathLineCoordinates.push([
          resource.x - 50,
          resource.parent_resource.y + 50,
        ]);
        this.healthTreePathLineCoordinates.push([
          resource.x - 50,
          resource.y + 50,
        ]);
        this.healthTreePathLineCoordinates.push(null);
        this.healthTreePathLineCoordinates.push([
          resource.x - 50,
          resource.y + this.resourceCardDim.headerHeight,
        ]);
        this.healthTreePathLineCoordinates.push([
          resource.x,
          resource.y + this.resourceCardDim.headerHeight,
        ]);
      }
      this.healthTreePathLineCoordinates.push(null);
    } else {
      resource.x = this.treeContainerDim.paddingX;
    }

    this.treeContainerDim.width =
      resource.x > this.treeContainerDim.width
        ? resource.x
        : this.treeContainerDim.width;

    HealthCardBuilder.buildConfig(resource);
    return resource;
  }

  private expandCollapse(data: any) {
    if (data._sub_resources) {
      // Expand
      data.sub_resources = data._sub_resources;
      data._sub_resources = null;
    } else {
      // Collapse
      data._sub_resources = data.sub_resources;
      data.sub_resources = [];
      d3.select("#nodeConnectorPath").remove();
    }
    this.resetTreeDimContainer();
    this.healthTreePathLineCoordinates = [];
    const nodes: any[] = this.traverse();
    this.buildSVG(nodes);
  }

  private resetTreeDimContainer() {
    this.treeContainerDim = {
      width: 0,
      height: 20,
      paddingX: 20,
      paddingY: 20,
    };
  }

  public handleExport() {
    switch (this.selectedExport) {
      case "SVG":
        downloadSVGAsText(".health-tree");
        break;
      case "PNG":
        downloadSVGAsPNG(".health-tree");
        break;
      case "JPEG":
        downloadSVGAsJPEG(".health-tree");
        break;
      case "PDF":
        downloadSVGAsPDF(".health-tree");
        break;
    }
  }
}
</script>
<style lang="scss" scoped>
.graphical-view-container {
  & > * {
    margin: 1.5em 0;
  }

  & > *:first-child {
    margin-left: 1.5rem;
  }

  .export-feature {
    display: flex;
    gap: 1em;
    max-width: 400px;
  }
}
.g_popup:focus {
  outline: none;
}
</style>
