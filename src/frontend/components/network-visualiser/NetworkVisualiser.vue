<template>
    <div class="flex-1 h-full min-w-full sm:min-w-[500px] relative dark:bg-zinc-950">
        <!-- network -->
        <div id="network" class="w-full h-full"></div>
        <!-- controls -->
        <div class="absolute flex bottom-0 left-0 bg-gray-100 dark:bg-zinc-900 p-2">
            <div class="bg-white dark:bg-zinc-800 rounded shadow min-w-52">
                <div @click="isShowingControls = !isShowingControls" class="flex text-gray-700 dark:text-gray-300 p-2 cursor-pointer">
                    <div class="my-auto">Reticulum Network</div>
                    <div class="flex ml-auto">
                        <button 
                            @click.stop="update" 
                            type="button" 
                            class="my-auto inline-flex items-center gap-x-1 rounded-md bg-gray-500 dark:bg-zinc-700 px-1 py-0.5 text-sm font-semibold text-white shadow-sm hover:bg-gray-400 dark:hover:bg-zinc-600 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-gray-500 dark:focus-visible:outline-zinc-600"
                        >
                            <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="size-5 text-white">
                                <path stroke-linecap="round" stroke-linejoin="round" d="M16.023 9.348h4.992v-.001M2.985 19.644v-4.992m0 0h4.992m-4.993 0 3.181 3.183a8.25 8.25 0 0 0 13.803-3.7M4.031 9.865a8.25 8.25 0 0 1 13.803-3.7l3.181 3.182m0-4.991v4.99" />
                            </svg>
                        </button>
                    </div>
                </div>
                <div v-if="isShowingControls" class="divide-y dark:divide-zinc-700 text-gray-900 dark:text-white border-t border-gray-300 dark:border-zinc-700">
                    <div class="px-1 py-2">
                        <div class="flex items-start">
                            <div class="flex items-center h-5">
                                <input 
                                    v-model="autoReload" 
                                    type="checkbox" 
                                    class="w-4 h-4 border border-gray-300 dark:border-zinc-600 rounded bg-gray-50 dark:bg-zinc-900 focus:ring-3 focus:ring-blue-300 dark:focus:ring-blue-800"
                                >
                            </div>
                            <label class="ml-2 text-sm font-medium text-gray-900 dark:text-white">Auto Update (5 sec)</label>
                        </div>
                    </div>
                    <div class="p-1">
                        <div class="text-black dark:text-white">Interfaces</div>
                        <div class="text-sm text-gray-700 dark:text-gray-300">{{ onlineInterfaces.length }} Online, {{ offlineInterfaces.length }} Offline</div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<style>
.vis-tooltip {
    color: white !important;
    background: rgba(0, 0, 0, 0.75) !important;
}

/* Force text color for vis-network labels in dark mode */
.dark .vis-network .vis-label {
    color: #ffffff !important;
    text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.8) !important;
}

/* Force text color for vis-network labels in light mode */
.vis-network .vis-label {
    color: #000000 !important;
    text-shadow: 1px 1px 2px rgba(255, 255, 255, 0.8) !important;
}

/* Ensure interface names are visible */
.dark .vis-network text {
    fill: #ffffff !important;
}

.vis-network text {
    fill: #000000 !important;
}
</style>

<script>
import "vis-network/styles/vis-network.css";
import { Network } from "vis-network";
import { DataSet } from "vis-data";
import Utils from "../../js/Utils";
import * as mdi from "@mdi/js";
export default {
    name: 'NetworkVisualiser',
    data() {
        return {
            config: null,
            autoReload: false,
            reloadInterval: null,
            isShowingControls: true,
            interfaces: [],
            pathTable: [],
            announces: {},
            network: null,
            nodes: new DataSet(),
            edges: new DataSet(),
        };
    },
    beforeUnmount() {
        clearInterval(this.reloadInterval);
    },
    mounted() {
        this.init();
    },
    methods: {
        async getInterfaceStats() {
            try {
                const response = await axios.get(`/api/v1/interface-stats`);
                this.interfaces = response.data.interface_stats?.interfaces ?? [];
            } catch(e) {
                console.log(e);
            }
        },
        async getPathTable() {
            try {
                const response = await axios.get(`/api/v1/path-table`);
                this.pathTable = response.data.path_table;
            } catch(e) {
                console.log(e);
            }
        },
        async getConfig() {
            try {
                const response = await axios.get("/api/v1/config");
                this.config = response.data.config;
            } catch(e) {
                console.error(e);
            }
        },
        async getAnnounces() {
            try {

                // fetch announces
                const response = await window.axios.get(`/api/v1/announces`);

                // cache announces
                this.announces = {};
                for(const announce of response.data.announces){
                    this.announces[announce.destination_hash] = announce;
                }

            } catch(e) {
                // do nothing if failed to load announces
                console.log(e);
            }
        },
        updateNetworkTheme() {
            if (!this.network) return;

            // Update network options with new theme colors
            this.network.setOptions({
                nodes: {
                    font: {
                        color: this.textColor,
                        background: this.textBackground,
                        size: 14,
                        face: 'arial',
                        strokeWidth: 1,
                        strokeColor: this.isDarkMode ? '#000000' : '#ffffff',
                    },
                },
                groups: {
                    "me": {
                        font: {
                            color: this.textColor,
                            background: this.textBackground,
                            size: 16,
                            face: 'arial',
                            strokeWidth: 1,
                            strokeColor: this.isDarkMode ? '#000000' : '#ffffff',
                        },
                    },
                    "interface": {
                        font: {
                            color: this.textColor,
                            background: this.textBackground,
                            size: 12,
                            face: 'arial',
                            strokeWidth: 1,
                            strokeColor: this.isDarkMode ? '#000000' : '#ffffff',
                        },
                    },
                    "announce": {
                        font: {
                            color: this.textColor,
                            background: this.textBackground,
                            size: 12,
                            face: 'arial',
                            strokeWidth: 1,
                            strokeColor: this.isDarkMode ? '#000000' : '#ffffff',
                        },
                    },
                },
            });

            // Force a redraw of all nodes
            this.network.redraw();
        },
        async init() {

            // create network ui
            const container = document.getElementById("network");
            this.network = new Network(container, {
                nodes: this.nodes,
                edges: this.edges,
            }, {
                interaction: {
                    tooltipDelay: 0, // show tooltip instantly on hover
                },
                layout: {
                    // always layout nodes the same way across reloads if nothing changed
                    randomSeed: 1,
                },
                nodes: {
                    color: {
                        border: "#000000",
                        highlight: {
                            border: "#000000",
                        },
                    },
                    font: {
                        color: this.textColor,
                        background: this.textBackground,
                        size: 14,
                        face: 'arial',
                        strokeWidth: 1,
                        strokeColor: this.isDarkMode ? '#000000' : '#ffffff',
                    },
                },
                physics: {
                    barnesHut: {
                        gravitationalConstant: -5000,
                        // centralGravity: 0,
                        // springConstant: 0.1,
                        // damping: 0.15,
                    },
                    // maxVelocity: 150,
                    // minVelocity: 0.25,
                },
                groups: {
                    "me": {
                        shape: "image",
                        image: "/assets/images/reticulum_logo_512.png",
                        font: {
                            color: this.textColor,
                            background: this.textBackground,
                            size: 16,
                            face: 'arial',
                            strokeWidth: 1,
                            strokeColor: this.isDarkMode ? '#000000' : '#ffffff',
                        },
                    },
                    "interface": {
                        font: {
                            color: this.textColor,
                            background: this.textBackground,
                            size: 12,
                            face: 'arial',
                            strokeWidth: 1,
                            strokeColor: this.isDarkMode ? '#000000' : '#ffffff',
                        },
                    },
                    "announce": {
                        font: {
                            color: this.textColor,
                            background: this.textBackground,
                            size: 12,
                            face: 'arial',
                            strokeWidth: 1,
                            strokeColor: this.isDarkMode ? '#000000' : '#ffffff',
                        },
                    },
                },
            });

            // handle double click on a node
            this.network.on("doubleClick", (params) => {

                // get clicked node id
                const clickedNodeId = params.nodes[0];
                if(!clickedNodeId){
                    return;
                }

                // find node by id
                const node = this.network.body.nodes[clickedNodeId];
                if(!node){
                    return;
                }

                // handle double click on an announce node
                if(node.options.group === "announce"){

                    // get announce
                    const announce = node.options._announce;
                    if(!announce) {
                        return;
                    }

                    // handle double click on lxmf.delivery node
                    if(announce.aspect === "lxmf.delivery"){

                        // go to messages page for this destination hash
                        this.$router.push({
                            name: "messages",
                            params: {
                                destinationHash: announce.destination_hash,
                            },
                        });

                    }

                    // handle double click on nomadnetwork.node node
                    if(announce.aspect === "nomadnetwork.node"){

                        // go to nomadnetwork page for this destination hash
                        this.$router.push({
                            name: "nomadnetwork",
                            params: {
                                destinationHash: announce.destination_hash,
                            },
                        });

                    }

                }

            });

            // update network
            await this.update();

            // update theme after initial load
            this.updateNetworkTheme();

            // fit network after initial load
            setTimeout(() => {
                this.network.fit({
                    animation: true,
                });
            }, 2000);

            // auto reload
            this.reloadInterval = setInterval(this.onAutoReload, 5000);

        },
        async onAutoReload() {

            // do nothing if auto reload disabled
            if(!this.autoReload){
                return;
            }

            // do nothing if already updating
            if(this.isUpdating){
                return;
            }

            // auto reload
            try {
                this.isUpdating = true;
                await this.update();
            } finally {
                this.isUpdating = false;
            }

        },
        async update() {

            await this.getConfig();
            await this.getInterfaceStats();
            await this.getPathTable();
            await this.getAnnounces();

            const nodes = [];
            const edges = [];

            // add me - enhanced GHOST GRID node
            const meNode = {
                id: "me",
                group: "me",
                size: 80,
                label: this.config?.display_name ?? "GHOST GRID",
                title: this.buildMyNodeTooltip(),
                font: {
                    color: this.textColor,
                    background: this.textBackground,
                    size: 18,
                    face: 'arial',
                    strokeWidth: 2,
                    strokeColor: this.isDarkMode ? '#000000' : '#ffffff',
                },
                shape: "box",
                borderWidth: 3,
                borderWidthSelected: 4,
                color: {
                    border: "#10b981", // emerald-500 for GHOST GRID theme
                    background: this.isDarkMode ? "#1f2937" : "#f9fafb", // gray-800 : gray-50
                    highlight: {
                        border: "#059669", // emerald-600
                        background: this.isDarkMode ? "#374151" : "#f3f4f6", // gray-700 : gray-100
                    },
                },
                shapeProperties: {
                    borderRadius: 12,
                },
                margin: 10,
            };

            // Add profile icon if available
            if (this.config?.lxmf_user_icon_name) {
                meNode.image = this.createProfileIconDataUrl();
                meNode.shape = "circularImage";
                meNode.size = 90;
                meNode.borderWidth = 4;
                meNode.color = {
                    border: "#10b981",
                    highlight: {
                        border: "#059669",
                    },
                };
            }

            nodes.push(meNode);

            // add interfaces
            for(const entry of this.interfaces){

                // determine label with better formatting
                var label = entry.interface_name ?? entry.name;

                // we want to show the full info for LocalServerInterface
                // we also want to show the full info instead of just "Client on Name" for TCPServerInterface clients
                if(entry.type === "LocalServerInterface" || entry.parent_interface_name != null){
                    label = entry.name;
                }

                // Clean up the label for better readability
                if (label) {
                    // Remove common prefixes and make it more readable
                    label = label.replace(/^Client on /, '');

                    // For interface names like "4c0ffe-8", add a more descriptive format
                    if (label.match(/^[a-f0-9]{6}-\d+$/)) {
                        const interfaceType = entry.type || 'Interface';
                        const shortType = interfaceType.replace('Interface', '').replace('Server', '');
                        label = `${shortType} ${label}`;
                    }

                    // Limit label length for better display
                    if (label.length > 20) {
                        label = label.substring(0, 17) + '...';
                    }
                }

                const node = {
                    id: entry.name,
                    group: "interface",
                    label: label,
                    title: [
                        entry.name,
                        `State: ${entry.status ? 'Online' : 'Offline'}`,
                        `Bitrate: ${this.formatBitsPerSecond(entry.bitrate)}`,
                        `TX: ${this.formatBytes(entry.txb)}`,
                        `RX: ${this.formatBytes(entry.rxb)}`,
                    ].join("\n"),
                    size: 30,
                    font: {
                        color: this.textColor,
                        background: this.textBackground,
                        size: 12,
                        face: 'arial',
                        strokeWidth: 1,
                        strokeColor: this.isDarkMode ? '#000000' : '#ffffff',
                    },
                    shape: "circularImage",
                    image: entry.status ? "/assets/images/network-visualiser/interface_connected.png" : "/assets/images/network-visualiser/interface_disconnected.png",
                };

                // add interface node
                nodes.push(node);

                // add edge to interface
                if(entry.parent_interface_name){
                    // add edge from parent interface to interface
                    edges.push({
                        id: `${entry.parent_interface_name}~${entry.name}`,
                        from: entry.parent_interface_name,
                        to: entry.name,
                        color: "transparent",
                        length: 300,
                        background: {
                            enabled: true,
                            color: entry.status ? "#22c55e" : "#ef4444",
                        },
                    });
                } else {
                    // add edge from me to interface
                    edges.push({
                        id: `me~${entry.name}`,
                        from: "me",
                        to: entry.name,
                        color: "transparent",
                        length: 300,
                        background: {
                            enabled: true,
                            color: entry.status ? "#22c55e" : "#ef4444",
                        },
                    });
                }

            }

            // add paths for announces
            for(const entry of this.pathTable){

                // skip this path if hops are unknown
                if(entry.hops == null){
                    continue;
                }

                // find what announced this path, or skip showing it for now
                const announce = this.announces[entry.hash];
                if(!announce){
                    continue;
                }

                // skip announces if we don't want to show them
                const aspectsToShow = ["lxmf.delivery", "nomadnetwork.node"];
                if(!aspectsToShow.includes(announce.aspect)){
                    continue;
                }

                const node = {
                    id: entry.hash,
                    group: "announce",
                    size: 20,
                };

                if(announce.aspect === "lxmf.delivery"){

                    const name = announce.custom_display_name ?? announce.display_name;

                    node.shape = "circularImage";
                    node.image = entry.hops === 1 ? "/assets/images/network-visualiser/user_1hop.png" : "/assets/images/network-visualiser/user.png";

                    node.label = name;
                    node.font = {
                        color: this.textColor,
                        background: this.textBackground,
                        size: 12,
                        face: 'arial',
                        strokeWidth: 1,
                        strokeColor: this.isDarkMode ? '#000000' : '#ffffff',
                    };
                    node.title = [
                        `Name: ${announce.display_name}`,
                        announce.custom_display_name != null ? `Custom Name: ${announce.custom_display_name}` : null,
                        `Aspect: ${announce.aspect}`,
                        `Identity: ${announce.identity_hash}`,
                        `Destination: ${announce.destination_hash}`,
                        `Path: ${entry.hops} ${entry.hops === 1 ? 'Hop' : 'Hops'} via ${entry.interface}`,
                        `Announced At: ${announce.updated_at}`,
                    ].filter((line) => line != null).join("\n");

                }

                if(announce.aspect === "nomadnetwork.node"){

                    const name = announce.custom_display_name ?? announce.display_name;

                    node.shape = "circularImage";
                    node.image = entry.hops === 1 ? "/assets/images/network-visualiser/server_1hop.png" : "/assets/images/network-visualiser/server.png";

                    node.label = name;
                    node.font = {
                        color: this.textColor,
                        background: this.textBackground,
                        size: 12,
                        face: 'arial',
                        strokeWidth: 1,
                        strokeColor: this.isDarkMode ? '#000000' : '#ffffff',
                    };
                    node.title = [
                        `Name: ${announce.display_name}`,
                        announce.custom_display_name != null ? `Custom Name: ${announce.custom_display_name}` : null,
                        `Aspect: ${announce.aspect}`,
                        `Identity: ${announce.identity_hash}`,
                        `Destination: ${announce.destination_hash}`,
                        `Path: ${entry.hops} ${entry.hops === 1 ? 'Hop' : 'Hops'} via ${entry.interface}`,
                        `Announced At: ${announce.updated_at}`,
                    ].filter((line) => line != null).join("\n");

                }

                // attach announce to this node
                node._announce = announce;

                // add node
                nodes.push(node);

                // add edge from interface to announced aspect
                edges.push({
                    id: `${entry.interface}~${entry.hash}`,
                    from: entry.interface,
                    to: entry.hash,
                    color: "gray",
                });

            }

            // process nodes and edges
            this.processNewNodes(nodes);
            this.processNewEdges(edges);

        },
        processNewNodes(newNodes) {

            // determine old and new nodes
            const oldNodeIds = this.nodes.map((node) => node.id);
            const newNodeIds = newNodes.map((node) => node.id);

            // log new nodes
            for(const newNodeId of newNodeIds){
                if(!oldNodeIds.includes(newNodeId)){
                    console.log("Added Node: " + newNodeId);
                }
            }

            // remove old nodes that no longer exist
            for(const oldNodeId of oldNodeIds){
                if(!newNodeIds.includes(oldNodeId)){
                    console.log("Removed Node: " + oldNodeId);
                    this.nodes.remove(oldNodeId);
                }
            }

            // update nodes
            this.nodes.update(newNodes);

        },
        processNewEdges(newEdges) {

            // determine old and new edges
            const oldEdgeIds = this.edges.map((edge) => edge.id);
            const newEdgeIds = newEdges.map((edge) => edge.id);

            // log new edges
            for(const newEdgeId of newEdgeIds){
                if(!oldEdgeIds.includes(newEdgeId)){
                    console.log("Added Edge: " + newEdgeId);
                }
            }

            // remove old edges that no longer exist
            for(const oldEdgeId of oldEdgeIds){
                if(!newEdgeIds.includes(oldEdgeId)){
                    console.log("Removed Edge: " + oldEdgeId);
                    this.edges.remove(oldEdgeId);
                }
            }

            // update edges
            this.edges.update(newEdges);

        },
        buildMyNodeTooltip() {
            if (!this.config) return "GHOST GRID Node";

            const lines = [
                `🔮 ${this.config.display_name || 'GHOST GRID'}`,
                ``,
                `📡 Communication Addresses:`,
                `Identity Hash: ${this.config.identity_hash || 'Unknown'}`,
                `LXMF Address: ${this.config.lxmf_address_hash || 'Unknown'}`,
            ];

            if (this.config.audio_call_address_hash) {
                lines.push(`Audio Call: ${this.config.audio_call_address_hash}`);
            }

            if (this.config.lxmf_local_propagation_node_address_hash) {
                lines.push(`Propagation Node: ${this.config.lxmf_local_propagation_node_address_hash}`);
            }

            lines.push(``);
            lines.push(`🌐 Status: ${this.config.is_transport_enabled ? 'Transport Node' : 'End Node'}`);

            if (this.config.auto_announce_enabled) {
                lines.push(`📢 Auto-announce: Every ${this.config.auto_announce_interval_seconds}s`);
            }

            return lines.join('\n');
        },
        createProfileIconDataUrl() {
            if (!this.config?.lxmf_user_icon_name) return null;

            const iconName = this.config.lxmf_user_icon_name;
            const foregroundColor = this.config.lxmf_user_icon_foreground_colour || '#6b7280';
            const backgroundColor = this.config.lxmf_user_icon_background_colour || '#e5e7eb';

            // Convert icon name from lxmf format to MDI format (same logic as MaterialDesignIcon.vue)
            const mdiIconName = "mdi" + iconName.split("-").map((word) => {
                return word.charAt(0).toUpperCase() + word.slice(1);
            }).join("");

            // Get the icon path from @mdi/js library
            const iconPath = mdi[mdiIconName] || mdi["mdiProgressQuestion"] || "";

            // If no valid icon path found, fallback to initials
            if (!iconPath) {
                const displayText = this.config.display_name ?
                    this.config.display_name.substring(0, 2).toUpperCase() :
                    iconName.substring(0, 2).toUpperCase();

                const svg = `
                    <svg xmlns="http://www.w3.org/2000/svg" width="80" height="80" viewBox="0 0 80 80">
                        <rect width="80" height="80" rx="12" fill="${backgroundColor}" stroke="#10b981" stroke-width="3"/>
                        <text x="40" y="50" text-anchor="middle" font-family="Arial, sans-serif" font-size="24" font-weight="bold" fill="${foregroundColor}">
                            ${displayText}
                        </text>
                    </svg>
                `;
                return 'data:image/svg+xml;base64,' + btoa(svg);
            }

            // Create SVG with actual MDI icon
            const svg = `
                <svg xmlns="http://www.w3.org/2000/svg" width="80" height="80" viewBox="0 0 80 80">
                    <rect width="80" height="80" rx="12" fill="${backgroundColor}" stroke="#10b981" stroke-width="3"/>
                    <svg x="16" y="16" width="48" height="48" viewBox="0 0 24 24" fill="${foregroundColor}">
                        <path d="${iconPath}"/>
                    </svg>
                </svg>
            `;

            // Convert to data URL
            return 'data:image/svg+xml;base64,' + btoa(svg);
        },
        formatBytes: function(bytes) {
            return Utils.formatBytes(bytes);
        },
        formatBitsPerSecond: function(bits) {
            return Utils.formatBitsPerSecond(bits);
        },
        updateNetworkTheme() {
            if (this.network) {
                // Update node font colors for theme change
                this.network.setOptions({
                    nodes: {
                        font: {
                            color: this.textColor,
                            background: this.textBackground,
                            strokeColor: this.isDarkMode ? '#000000' : '#ffffff',
                        },
                    },
                });

                // Trigger a redraw
                this.network.redraw();
            }
        },
    },
    computed: {
        onlineInterfaces() {
            return this.interfaces.filter((iface) => {
                return iface.status;
            });
        },
        offlineInterfaces() {
            return this.interfaces.filter((iface) => {
                return !iface.status;
            });
        },
        isDarkMode() {
            return this.config?.theme === 'dark';
        },
        textColor() {
            return this.isDarkMode ? '#ffffff' : '#000000';
        },
        textBackground() {
            return this.isDarkMode ? 'rgba(0, 0, 0, 0.7)' : '#ffffff';
        },
    },
    watch: {
        isDarkMode() {
            // Update network theme when dark mode changes
            this.$nextTick(() => {
                this.updateNetworkTheme();
            });
        },
    },
}
</script>
