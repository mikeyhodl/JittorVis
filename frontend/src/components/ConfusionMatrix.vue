<template>
    <svg id="confusion-svg" width="100%" height="100%" ref="svg">
        <g id="main-g" transform="translate(0,0)">
            <g id="legend-g" :transform="`translate(5,${leftCornerSize/2-25})`"></g>
            <g id="horizon-text-g" :transform="`translate(${leftCornerSize-maxHorizonTextWidth}, ${leftCornerSize+textMatrixMargin})`"></g>
            <g id="vertical-text-g" :transform="`translate(${leftCornerSize+textMatrixMargin}, ${leftCornerSize}) rotate(-90)`"></g>
            <g id="matrix-cells-g" :transform="`translate(${leftCornerSize+textMatrixMargin}, ${leftCornerSize+textMatrixMargin})`"></g>
        </g>
    </svg>
</template>

<script>
import {mapGetters} from 'vuex';
import * as d3 from 'd3';
import Util from './Util.vue';
import GlobalVar from './GlovalVar.vue';
import clone from 'just-clone';
import {Mutex} from 'async-mutex';

export default {
    name: 'ConfusionMatrix',
    mixins: [Util, GlobalVar],
    props: {
        showColor: {
            type: Boolean,
            default: false,
        },
    },
    computed: {
        ...mapGetters([
            'confusionMatrix',
            'labelHierarchy',
            'labelnames',
            'hierarchyColors',
            'colors',
            'shownClass',
        ]),
        baseMatrix: function() {
            return this.confusionMatrix;
        },
        indexNames: function() {
            return this.labelnames;
        },
        rawHierarchy: function() {
            return this.labelHierarchy;
        },
        name2index: function() {
            const result = {};
            for (let i=0; i<this.indexNames.length; i++) {
                result[this.indexNames[i]] = i;
            }
            return result;
        },
        svg: function() {
            return d3.select('#confusion-svg');
        },
        matrixWidth: function() {
            return this.showNodes.length * this.cellAttrs['size'];
        },
        svgWidth: function() {
            return this.leftCornerSize+this.textMatrixMargin+this.matrixWidth;
        },
        colorCellSize: function() {
            return this.showColor?this.cellAttrs['size']*0.7:0;
        },
        colorCellMargin: function() {
            return this.showColor?10:0;
        },
        horizonTextG: function() {
            return d3.select('g#horizon-text-g');
        },
        verticalTextG: function() {
            return d3.select('g#vertical-text-g');
        },
        matrixCellsG: function() {
            return d3.select('g#matrix-cells-g');
        },
        legendG: function() {
            return d3.select('g#legend-g');
        },
        mainG: function() {
            return d3.selectAll('g#main-g');
        },
        maxHorizonTextWidth: function() {
            let maxwidth = 0;
            for (const node of this.showNodes) {
                const textwidth = this.getTextWidth(node.name,
                    `${this.horizonTextAttrs['font-weight']} ${this.horizonTextAttrs['font-size']}px ${this.horizonTextAttrs['font-family']}`);
                const arrowIconNum = node.children.length===0?node.depth:node.depth+1;
                maxwidth = Math.max(maxwidth, this.horizonTextAttrs['leftMargin']*node.depth + textwidth +
                    arrowIconNum*(this.horizonTextAttrs['font-size'] + this.horizonTextAttrs['iconMargin'])+this.colorCellSize+this.colorCellMargin);
            }
            return maxwidth;
        },
        legendWidth: function() {
            return Math.max(100, this.maxHorizonTextWidth);
        },
        leftCornerSize: function() {
            return this.legendWidth;
        },
        colorScale: function() {
            return d3.scaleSequential([0, this.maxCellValue], ['rgb(255, 255, 255)', 'rgb(8, 48, 107)']).clamp(true);
        },
    },
    mounted: function() {
        this.hierarchy = this.getHierarchy(this.rawHierarchy);
        this.getDataAndRender();
    },
    watch: {
        labelHierarchy: function(newLabelHierarchy, oldLabelHierarchy) {
            this.hierarchy = this.getHierarchy(newLabelHierarchy);
            this.getDataAndRender();
        },
        hierarchyColors: function() {
            this.getDataAndRender();
        },
        shownClass: function(newShownClass, oldShownClass) {
            this.updateHierarchy(newShownClass);
            this.getDataAndRender();
        },
    },
    data: function() {
        return {
            hierarchy: {},
            // layout
            textGWidth: 0,
            cellWidth: 10,
            textMatrixMargin: 10,
            showNodes: [],
            cells: [],
            // layout elements
            horizonTextinG: null,
            verticalTextinG: null,
            matrixCellsinG: null,
            // render attrs
            horizonTextAttrs: {
                'gClass': 'horizon-one-line-g',
                'leftMargin': 30,
                'text-anchor': 'start',
                'font-family': 'Comic Sans MS',
                'font-weight': 'normal',
                'font-size': 15,
                'iconMargin': 5,
                'iconDy': 3,
                'indent-line-stroke': 'gray',
                'indent-line-stroke-width': 2,
            },
            verticalTextAttrs: {
                'gClass': 'vertical-one-line-g',
                'leftMargin': 30,
                'text-anchor': 'start',
                'font-family': 'Comic Sans MS',
                'font-weight': 'normal',
                'font-size': 15,
                'iconMargin': 5,
                'iconDy': 3,
                'indent-line-stroke': 'gray',
                'indent-line-stroke-width': 2,
            },
            cellAttrs: {
                'gClass': 'one-cell-g',
                'size': 30,
                'stroke-width': '1px',
                'stroke': 'gray',
                'slash-text-stroke': 'gray',
                'text-fill': '#FF6A6A',
                'text-anchor': 'start',
                'font-family': 'Comic Sans MS',
                'font-weight': 'normal',
                'font-size': 15,
                'cursor': 'pointer',
            },
            // buffer
            maxCellValue: 0,
            mutex: new Mutex(),
        };
    },
    methods: {
        highlightCell: function(node) {
            const label = node[0];
            const pred = node[1];
            let shownLabel = '';
            let shownPred = '';
            const findShownLabel = function(label, root, parents) {
                if (root.name===label) {
                    if (parents.length===0) {
                        return label;
                    } else {
                        for (const p of parents) {
                            if (p.expand===false) {
                                return p.name;
                            }
                        }
                        return label;
                    }
                }
                parents.push({name: root.name, expand: root.expand});
                for (const child of root.children) {
                    const result = findShownLabel(label, child, parents);
                    if (result!==null) {
                        return result;
                    }
                }
                parents.pop();
                return null;
            };
            for (const child of this.hierarchy) {
                const resultLabel = findShownLabel(label, child, []);
                if (resultLabel) shownLabel = resultLabel;
            }
            for (const child of this.hierarchy) {
                const resultPred = findShownLabel(pred, child, []);
                if (resultPred) shownPred = resultPred;
            }

            // console.log(label, pred, shownLabel+','+shownPred);
            const that = this;
            d3.selectAll('.one-cell-g').filter((d) => d.key === shownLabel+','+shownPred)
                .each(function(d, v, x) {
                    // d3.select(x[0]).select('rect').attr('fill', 'red');
                    const strokeWidth = 4;
                    d3.select(x[0])
                        .append('rect')
                        .attr('class', 'mask')
                        .attr('x', strokeWidth / 2)
                        .attr('y', strokeWidth / 2)
                        .attr('width', that.cellAttrs['size'] - strokeWidth)
                        .attr('height', that.cellAttrs['size'] - strokeWidth)
                        .attr('stroke', 'orange')
                        .attr('stroke-width', strokeWidth)
                        .attr('fill', 'none');
                });
        },
        unhighlightCell(node) {
            d3.selectAll('.one-cell-g').selectAll('.mask').remove();
        },
        getHierarchy: function(hierarchy) {
            hierarchy = clone(hierarchy);
            const postorder = function(root, depth) {
                if (typeof(root) !== 'object') {
                    return {
                        name: root,
                        expand: false,
                        leafs: [root],
                        children: [],
                        depth: depth,
                    };
                }
                root.expand = false;
                root.depth = depth;
                let leafs = [];
                const newChildren = [];
                for (const child of root.children) {
                    const newChild = postorder(child, depth+1);
                    leafs = leafs.concat(newChild.leafs);
                    newChildren.push(newChild);
                }
                root.children = newChildren;
                root.leafs = leafs;
                return root;
            };
            for (const root of Object.values(hierarchy)) {
                postorder(root, 0);
            }
            return hierarchy;
        },
        updateHierarchy: function(shownClass) {
            console.log('hierarchy', shownClass, this.hierarchy);
            const travel = function(root) {
                root.expand = false;
                for (const child of root.children) {
                    if (shownClass.indexOf(child.name) !== -1) {
                        root.expand = true;
                        break;
                    }
                }
                if (root.expand) {
                    for (const child of root.children) {
                        travel(child);
                    }
                } else {
                    setUnexpand(root);
                }
            };
            const setUnexpand = function(root) {
                if (!root) {
                    return;
                }
                root.expand = false;
                for (const child of root.children) {
                    setUnexpand(child);
                }
            };
            for (const child of this.hierarchy) {
                travel(child);
            }
        },
        getShowNodes: function(hierarchy) {
            const showNodes = [];
            const stack = Object.values(hierarchy).reverse();
            while (stack.length>0) {
                const top = stack.pop();
                showNodes.push(top);
                if (top.expand) {
                    for (let i=top.children.length-1; i>=0; i--) {
                        stack.push(top.children[i]);
                    }
                }
            }
            return showNodes;
        },
        getDataAndRender: function() {
            const that = this;
            this.mutex.runExclusive(function() {
                // this.setLabelColorsByHierarchy(this.colors, this.hierarchy);
            // get nodes to show
                that.showNodes = that.getShowNodes(that.hierarchy);
                // get cells to render
                that.cells = [];
                that.maxCellValue = 0;
                for (let i=0; i<that.showNodes.length; i++) {
                    const nodea = that.showNodes[i];
                    for (let j=0; j<that.showNodes.length; j++) {
                        const nodeb = that.showNodes[j];
                        const cell = {
                            key: nodea.name+','+nodeb.name,
                            value: that.getTwoCellConfusion(nodea, nodeb),
                            row: i,
                            column: j,
                            rowNode: nodea,
                            colNode: nodeb,
                        };
                        that.cells.push(cell);
                        if (!that.isHideCell(cell) && i!=j) {
                            that.maxCellValue = Math.max(that.maxCellValue, cell.value*2);
                        }
                    }
                }
                that.render();
            });
        },
        setLabelColorsByHierarchy: function(colors, hierarchy) {
            const hierarchyColors = {};
            const dfs = function(root, colors, hierarchyColors, isExpand, parentColor) {
                if (isExpand) {
                    hierarchyColors[root.name] = colors[root.name];
                } else {
                    hierarchyColors[root.name] = parentColor;
                }
                for (const child of root.children) {
                    dfs(child, colors, hierarchyColors, isExpand&&root.expand, hierarchyColors[root.name]);
                }
            };
            for (const root of hierarchy) {
                dfs(root, colors, hierarchyColors, true, '');
            }
            this.$store.commit('setHierarchyColors', hierarchyColors);
        },
        render: async function() {
            this.horizonTextinG = this.horizonTextG.selectAll('g.'+this.horizonTextAttrs['gClass']).data(this.showNodes, (d)=>d.name);
            this.verticalTextinG = this.verticalTextG.selectAll('g.'+this.verticalTextAttrs['gClass']).data(this.showNodes, (d)=>d.name);
            this.matrixCellsinG = this.matrixCellsG.selectAll('g.'+this.cellAttrs['gClass']).data(this.cells, (d)=>d.key);

            this.drawLegend();
            await this.remove();
            await this.update();
            await this.transform();
            await this.create();
        },
        create: async function() {
            const that = this;
            return new Promise((resolve, reject) => {
                const horizonTextinG = that.horizonTextinG.enter()
                    .append('g')
                    .attr('class', that.horizonTextAttrs['gClass'])
                    .attr('opacity', 0)
                    .attr('transform', (d, i) => `translate(${d.depth*that.horizonTextAttrs['leftMargin']}, 
                        ${i*that.cellAttrs['size']})`);

                horizonTextinG.transition()
                    .duration(that.createDuration)
                    .attr('opacity', 1)
                    .on('end', resolve);

                horizonTextinG.append('text')
                    .attr('x', (d) => (d.children.length===0?0:that.horizonTextAttrs['font-size'] +
                        that.horizonTextAttrs['iconMargin'])+that.colorCellSize + that.colorCellMargin)
                    .attr('y', 0)
                    .attr('dy', that.cellAttrs['size']/2+that.horizonTextAttrs['font-size']/2)
                    .attr('text-anchor', that.horizonTextAttrs['text-anchor'])
                    .attr('font-size', that.horizonTextAttrs['font-size'])
                    .attr('font-weight', that.horizonTextAttrs['font-weight'])
                    .attr('font-family', that.horizonTextAttrs['font-family'])
                    .text((d) => d.name);

                const icony = that.cellAttrs['size']/2-that.horizonTextAttrs['font-size']/2+that.horizonTextAttrs['iconDy'];
                horizonTextinG.filter((d) => d.children.length>0)
                    .append('image')
                    .attr('xlink:href', '/static/images/arrow.png')
                    .attr('x', 0)
                    .attr('y', icony)
                    .attr('width', that.horizonTextAttrs['font-size'])
                    .attr('height', that.horizonTextAttrs['font-size'])
                    .attr('transform', (d) => `rotate(${d.expand?90:0} 
                        ${that.horizonTextAttrs['font-size']/2} ${icony+that.horizonTextAttrs['font-size']/2})`)
                    .attr('cursor', 'pointer')
                    .on('click', function(e, d) {
                        d.expand = !d.expand;
                        that.getDataAndRender();
                    });

                horizonTextinG.filter((d) => d.children.length>0)
                    .append('path')
                    .attr('stroke', that.horizonTextAttrs['indent-line-stroke'])
                    .attr('stroke-width', that.horizonTextAttrs['indent-line-stroke-width'])
                    .attr('d', (d)=>{
                        // find expand child length
                        const stack = [d];
                        let expandlen = 0;
                        if (d.expand) {
                            while (stack.length > 0) {
                                const top = stack.pop();
                                for (const child of top.children) {
                                    if (child.children.length>0 && child.expand===true) {
                                        stack.push(child);
                                    }
                                    expandlen++;
                                }
                            }
                        }
                        const linelen = that.cellAttrs['size']*expandlen;
                        const x = that.horizonTextAttrs['font-size']/2;
                        return `M ${x} ${that.cellAttrs['size']} L ${x} ${that.cellAttrs['size']+linelen}`;
                    });

                const verticalTextinG = that.verticalTextinG.enter()
                    .append('g')
                    .attr('class', that.verticalTextAttrs['gClass'])
                    .attr('opacity', 0)
                    .attr('transform', (d, i) => `translate(${d.depth*that.verticalTextAttrs['leftMargin']}, 
                        ${i*that.cellAttrs['size']})`);

                verticalTextinG.transition()
                    .duration(that.createDuration)
                    .attr('opacity', 1)
                    .on('end', resolve);

                verticalTextinG.append('text')
                    .attr('x', (d) => (d.children.length===0?0:that.verticalTextAttrs['font-size'] +
                        that.verticalTextAttrs['iconMargin'])+that.colorCellSize + that.colorCellMargin)
                    .attr('y', 0)
                    .attr('dy', that.cellAttrs['size']/2+that.horizonTextAttrs['font-size']/2)
                    .attr('text-anchor', that.verticalTextAttrs['text-anchor'])
                    .attr('font-size', that.verticalTextAttrs['font-size'])
                    .attr('font-weight', that.verticalTextAttrs['font-weight'])
                    .attr('font-family', that.verticalTextAttrs['font-family'])
                    .text((d) => d.name);

                verticalTextinG.filter((d) => d.children.length>0)
                    .append('image')
                    .attr('xlink:href', '/static/images/arrow.png')
                    .attr('x', 0)
                    .attr('y', icony)
                    .attr('width', that.verticalTextAttrs['font-size'])
                    .attr('height', that.verticalTextAttrs['font-size'])
                    .attr('transform', (d) => `rotate(${d.expand?90:0} 
                        ${that.verticalTextAttrs['font-size']/2} ${icony+that.verticalTextAttrs['font-size']/2})`)
                    .attr('cursor', 'pointer')
                    .on('click', function(e, d) {
                        d.expand = !d.expand;
                        that.getDataAndRender();
                    });

                verticalTextinG.filter((d) => d.children.length>0)
                    .append('path')
                    .attr('stroke', that.verticalTextAttrs['indent-line-stroke'])
                    .attr('stroke-width', that.verticalTextAttrs['indent-line-stroke-width'])
                    .attr('d', (d)=>{
                        // find expand child length
                        const stack = [d];
                        let expandlen = 0;
                        if (d.expand) {
                            while (stack.length > 0) {
                                const top = stack.pop();
                                for (const child of top.children) {
                                    if (child.children.length>0 && child.expand===true) {
                                        stack.push(child);
                                    }
                                    expandlen++;
                                }
                            }
                        }
                        const linelen = that.cellAttrs['size']*expandlen;
                        const x = that.verticalTextAttrs['font-size']/2;
                        return `M ${x} ${that.cellAttrs['size']} L ${x} ${that.cellAttrs['size']+linelen}`;
                    });

                if (that.showColor) {
                    horizonTextinG.append('rect')
                        .attr('x', (d) => d.children.length===0?0:that.horizonTextAttrs['font-size'] + that.horizonTextAttrs['iconMargin'])
                        .attr('y', (that.cellAttrs['size']-that.colorCellSize)/2+that.horizonTextAttrs['iconDy'])
                        .attr('width', that.colorCellSize)
                        .attr('height', that.colorCellSize)
                        .attr('stroke', that.cellAttrs['stroke'])
                        .attr('stroke-width', that.cellAttrs['stroke-width'])
                        .attr('fill', (d) => that.hierarchyColors[d.name].fill)
                        .attr('opacity', (d) => that.hierarchyColors[d.name].opacity);

                    verticalTextinG.append('rect')
                        .attr('x', (d) => d.children.length===0?0:that.verticalTextAttrs['font-size'] + that.verticalTextAttrs['iconMargin'])
                        .attr('y', (that.cellAttrs['size']-that.colorCellSize)/2+that.verticalTextAttrs['iconDy'])
                        .attr('width', that.colorCellSize)
                        .attr('height', that.colorCellSize)
                        .attr('stroke', that.cellAttrs['stroke'])
                        .attr('stroke-width', that.cellAttrs['stroke-width'])
                        .attr('fill', (d) => that.hierarchyColors[d.name].fill)
                        .attr('opacity', (d) => that.hierarchyColors[d.name].opacity);
                }

                const matrixCellsinG = that.matrixCellsinG.enter()
                    .append('g')
                    .attr('class', that.cellAttrs['gClass'])
                    .attr('opacity', 0)
                    .attr('cursor', that.cellAttrs['cursor'])
                    .attr('transform', (d) => `translate(${d.column*that.cellAttrs['size']}, 
                        ${d.row*that.cellAttrs['size']})`)
                    .on('click', function(e, d) {
                        that.$emit('clickCell', d);
                    });

                matrixCellsinG.transition()
                    .duration(that.createDuration)
                    .attr('opacity', (d)=>(that.isHideCell(d)?0:1))
                    .on('end', resolve);

                matrixCellsinG.append('rect')
                    .attr('x', 0)
                    .attr('y', 0)
                    .attr('width', that.cellAttrs['size'])
                    .attr('height', that.cellAttrs['size'])
                    .attr('stroke', that.cellAttrs['stroke'])
                    .attr('stroke-width', that.cellAttrs['stroke-width'])
                    .attr('fill', (d)=>that.colorScale(d.value));

                matrixCellsinG.filter((d) => d.value===0)
                    .append('path')
                    .attr('d', `M ${that.cellAttrs['size']*0.25} ${that.cellAttrs['size']*0.25} 
                        L ${that.cellAttrs['size']*0.75} ${that.cellAttrs['size']*0.75}`)
                    .attr('stroke', that.cellAttrs['slash-text-stroke']);

                matrixCellsinG.filter((d) => d.value>0)
                    .append('text')
                    .attr('x', that.cellAttrs['size']/2)
                    .attr('y', (that.cellAttrs['size']+that.cellAttrs['font-size'])/2)
                    .attr('text-anchor', 'middle')
                    .attr('font-size', that.cellAttrs['font-size'])
                    .attr('font-weight', that.cellAttrs['font-weight'])
                    .attr('font-family', that.cellAttrs['font-family'])
                    .attr('opacity', 0)
                    .attr('fill', that.cellAttrs['text-fill'])
                    .text((d) => d.value);


                if ((that.horizonTextinG.enter().size() === 0) && (that.verticalTextinG.enter().size() === 0) &&
                    (that.matrixCellsinG.enter().size() === 0)) {
                    resolve();
                }
            });
        },
        update: async function() {
            const that = this;
            return new Promise((resolve, reject) => {
                that.horizonTextinG
                    .transition()
                    .duration(that.updateDuration)
                    .attr('transform', (d, i) => `translate(${d.depth*that.horizonTextAttrs['leftMargin']}, 
                        ${i*that.cellAttrs['size']})`)
                    .on('end', resolve);

                const icony = that.cellAttrs['size']/2-that.horizonTextAttrs['font-size']/2+that.horizonTextAttrs['iconDy'];
                that.horizonTextinG.filter((d) => d.children.length>0)
                    .selectAll('image')
                    .attr('transform', (d) => `rotate(${d.expand?90:0} 
                        ${that.horizonTextAttrs['font-size']/2} ${icony+that.horizonTextAttrs['font-size']/2})`);

                that.horizonTextinG.filter((d) => d.children.length>0)
                    .selectAll('path')
                    .attr('stroke', that.horizonTextAttrs['indent-line-stroke'])
                    .attr('stroke-width', that.horizonTextAttrs['indent-line-stroke-width'])
                    .transition()
                    .duration(that.updateDuration)
                    .attr('d', (d)=>{
                        // find expand child length
                        const stack = [d];
                        let expandlen = 0;
                        if (d.expand) {
                            while (stack.length > 0) {
                                const top = stack.pop();
                                for (const child of top.children) {
                                    if (child.children.length>0 && child.expand===true) {
                                        stack.push(child);
                                    }
                                    expandlen++;
                                }
                            }
                        }
                        const linelen = that.cellAttrs['size']*expandlen;
                        const x = that.horizonTextAttrs['font-size']/2;
                        return `M ${x} ${that.cellAttrs['size']} L ${x} ${that.cellAttrs['size']+linelen}`;
                    });

                that.verticalTextinG
                    .transition()
                    .duration(that.updateDuration)
                    .attr('transform', (d, i) => `translate(${d.depth*that.verticalTextAttrs['leftMargin']}, 
                        ${i*that.cellAttrs['size']})`)
                    .on('end', resolve);

                that.verticalTextinG.filter((d) => d.children.length>0)
                    .selectAll('image')
                    .attr('transform', (d) => `rotate(${d.expand?90:0} 
                        ${that.verticalTextAttrs['font-size']/2} ${icony+that.verticalTextAttrs['font-size']/2})`);

                that.verticalTextinG.filter((d) => d.children.length>0)
                    .selectAll('path')
                    .attr('stroke', that.verticalTextAttrs['indent-line-stroke'])
                    .attr('stroke-width', that.verticalTextAttrs['indent-line-stroke-width'])
                    .transition()
                    .duration(that.updateDuration)
                    .attr('d', (d)=>{
                        // find expand child length
                        const stack = [d];
                        let expandlen = 0;
                        if (d.expand) {
                            while (stack.length > 0) {
                                const top = stack.pop();
                                for (const child of top.children) {
                                    if (child.children.length>0 && child.expand===true) {
                                        stack.push(child);
                                    }
                                    expandlen++;
                                }
                            }
                        }
                        const linelen = that.cellAttrs['size']*expandlen;
                        const x = that.verticalTextAttrs['font-size']/2;
                        return `M ${x} ${that.cellAttrs['size']} L ${x} ${that.cellAttrs['size']+linelen}`;
                    });

                if (that.showColor) {
                    that.horizonTextinG.selectAll('rect')
                        .attr('x', (d) => d.children.length===0?0:that.horizonTextAttrs['font-size'] + that.horizonTextAttrs['iconMargin'])
                        .attr('y', (that.cellAttrs['size']-that.colorCellSize)/2+that.horizonTextAttrs['iconDy'])
                        .attr('fill', (d) => that.hierarchyColors[d.name].fill)
                        .attr('opacity', (d) => that.hierarchyColors[d.name].opacity);

                    that.verticalTextinG.selectAll('rect')
                        .attr('x', (d) => d.children.length===0?0:that.verticalTextAttrs['font-size'] + that.verticalTextAttrs['iconMargin'])
                        .attr('y', (that.cellAttrs['size']-that.colorCellSize)/2+that.verticalTextAttrs['iconDy'])
                        .attr('fill', (d) => that.hierarchyColors[d.name].fill)
                        .attr('opacity', (d) => that.hierarchyColors[d.name].opacity);
                }

                that.matrixCellsinG
                    .transition()
                    .duration(that.updateDuration)
                    .attr('opacity', (d)=>(that.isHideCell(d)?0:1))
                    .attr('transform', (d) => `translate(${d.column*that.cellAttrs['size']}, 
                        ${d.row*that.cellAttrs['size']})`)
                    .on('end', resolve);

                that.matrixCellsinG.selectAll('rect')
                    .transition()
                    .duration(that.updateDuration)
                    .attr('fill', (d)=>that.colorScale(d.value))
                    .on('end', resolve);

                if ((that.horizonTextinG.size() === 0) && (that.verticalTextinG.size() === 0) &&
                    (that.matrixCellsinG.size() === 0)) {
                    resolve();
                }
            });
        },
        remove: async function() {
            const that = this;
            return new Promise((resolve, reject) => {
                that.horizonTextinG.exit()
                    .transition()
                    .duration(that.removeDuration)
                    .attr('opacity', 0)
                    .remove()
                    .on('end', resolve);

                that.verticalTextinG.exit()
                    .transition()
                    .duration(that.removeDuration)
                    .attr('opacity', 0)
                    .remove()
                    .on('end', resolve);

                that.matrixCellsinG.exit()
                    .transition()
                    .duration(that.removeDuration)
                    .attr('opacity', 0)
                    .remove()
                    .on('end', resolve);

                if ((that.horizonTextinG.exit().size() === 0) && (that.verticalTextinG.exit().size() === 0) &&
                    (that.matrixCellsinG.exit().size() === 0)) {
                    resolve();
                }
            });
        },
        transform: async function() {
            const that = this;
            return new Promise((resolve, reject) => {
                // compute transform
                const svgRealWidth = that.$refs.svg.clientWidth;
                const svgRealHeight = that.$refs.svg.clientHeight;
                const realSize = Math.min(svgRealWidth, svgRealHeight);
                let shiftx = 0;
                let shifty = 0;
                let scale = 1;
                if (that.svgWidth > realSize) {
                    scale = realSize/that.svgWidth;
                } else {
                    scale = 1;
                }
                shiftx = (svgRealWidth-scale*that.svgWidth)/2;
                shifty = (svgRealHeight-scale*that.svgWidth)/2;
                that.mainG.transition()
                    .duration(that.transformDuration)
                    .attr('transform', `translate(${shiftx} ${shifty}) scale(${scale})`)
                    .on('end', resolve);
            });
        },
        getTwoCellConfusion: function(nodea, nodeb) {
            let cnt = 0;
            for (const leafa of nodea.leafs) {
                for (const leafb of nodeb.leafs) {
                    cnt += this.baseMatrix[this.name2index[leafa]][this.name2index[leafb]];
                }
            }
            return cnt;
        },
        drawLegend: function() {
            const that = this;
            // https://observablehq.com/@d3/color-legend
            const drawLegend = function({
                color,
                title,
                tickSize = 6,
                width = 320,
                height = 44 + tickSize,
                marginTop = 18,
                marginRight = 0,
                marginBottom = 16 + tickSize,
                marginLeft = 0,
                ticks = width / 64,
                tickFormat,
                tickValues,
            } = {}) {
                const ramp = function(color, n = 256) {
                    const canvas = that.drawLegend.canvas || (that.drawLegend.canvas = document.createElement('canvas'));
                    canvas.width = n;
                    canvas.height = 1;
                    const context = canvas.getContext('2d');
                    for (let i = 0; i < n; ++i) {
                        context.fillStyle = color(i / (n - 1));
                        context.fillRect(i, 0, 1, 1);
                    }
                    return canvas;
                };
                const tickAdjust = (g) => g.selectAll('.tick line').attr('y1', marginTop + marginBottom - height);
                let x;

                // Continuous
                if (color.interpolator) {
                    x = Object.assign(color.copy()
                        .interpolator(d3.interpolateRound(marginLeft, width - marginRight)),
                    {range() {
                        return [marginLeft, width - marginRight];
                    }});

                    that.legendG.append('image')
                        .attr('x', marginLeft)
                        .attr('y', marginTop)
                        .attr('width', width - marginLeft - marginRight)
                        .attr('height', height - marginTop - marginBottom)
                        .attr('preserveAspectRatio', 'none')
                        .attr('xlink:href', ramp(color.interpolator()).toDataURL());

                    // scaleSequentialQuantile doesn’t implement ticks or tickFormat.
                    if (!x.ticks) {
                        if (tickValues === undefined) {
                            const n = Math.round(ticks + 1);
                            tickValues = d3.range(n).map((i) => d3.quantile(color.domain(), i / (n - 1)));
                        }
                        if (typeof tickFormat !== 'function') {
                            tickFormat = d3.format(tickFormat === undefined ? ',f' : tickFormat);
                        }
                    }
                }
                that.legendG.append('g')
                    .attr('transform', `translate(0,${height - marginBottom})`)
                    .call(d3.axisBottom(x)
                        .ticks(ticks, typeof tickFormat === 'string' ? tickFormat : undefined)
                        .tickFormat(typeof tickFormat === 'function' ? tickFormat : undefined)
                        .tickSize(tickSize)
                        .tickValues(tickValues))
                    .call(tickAdjust)
                    .call((g) => g.select('.domain').remove())
                    .call((g) => g.append('text')
                        .attr('x', marginLeft)
                        .attr('y', marginTop + marginBottom - height - 6)
                        .attr('fill', 'currentColor')
                        .attr('text-anchor', 'start')
                        .attr('font-weight', 'bold')
                        .attr('class', 'title')
                        .text(title));

                return that.legendG.node();
            };
            this.legendG.selectAll('*').remove();
            drawLegend(
                {
                    color: this.colorScale,
                    title: 'Counts',
                    width: this.legendWidth,
                    ticks: 5,
                },
            );
        },
        isHideCell: function(cell) {
            const isHideNode = function(node) {
                return node.expand===true && node.children.length>0;
            };
            return isHideNode(this.showNodes[cell.row]) || isHideNode(this.showNodes[cell.column]);
        },
    },
};
</script>
