<script>
	import * as d3 from "d3";
	import { onMount } from "svelte";
	import { imagesEndpoint } from "../../stores/endPoints";
	import {
		selectedParent,
		selectedImage,
		showMisclassifications,
		nodeHovering,
		treemapImageSize,
		treemapNumClusters,
		highlightSimilarImages,
		changingSizes,
		hideLabelAccuracy,
		hideLabelCoverage,
		highlightIncorrectImages,
		hideMisclassifiedImages,
	} from "../../stores/sidebarStore";
	import {
		globalClasses,
		globalRootNode,
		correctColor,
		incorrectColor,
	} from "../../stores/globalDataStore";
	import {
		treemapColorGenerator,
		toPercent,
		classCountsLabel,
		ID,
		forEachSelection,
	} from "./util";
	import { highlightImages, resetOpacity } from "./highlightImages";
	import { kClustersTreeMap } from "./treemapper";

	// global values used and renamed for this file
	/** @type {HierarchyNode} */
	$: data = $globalRootNode;
	$: imageWidth = $treemapImageSize;
	$: imageHeight = $treemapImageSize;
	$: numClusters = $treemapNumClusters;

	/**
	 *  These are for the jsdoc for better autocomplete for when doing type comments
	 * @typedef {{isLeaf: boolean, x0: number, y0: number, x1: number, y1: number}} AdditionalProperties
	 * @typedef {(d3.HierarchyNode | AdditionalProperties} HierarchyNode
	 */

	// style and dimensions
	export let width = 1600;
	export let height = 1000;
	export let svgStyle = "";
	export let transitionSpeed = 750;
	export let innerPadding = 0;
	export let topPadding = 0;
	export let highlightedOpacity = 1.0;
	export let hiddenOpacity = 0.25;
	export let topLabelSpace = 20;

	/**@type {d3.Selection}*/
	let group;
	/**@type {d3.Selection}*/
	let svg;
	/**@type {(t: number) => string}*/
	let color;
	/** @type {HierarchyNode[]} */
	let prevRoots = []; // stack used for keep track of last root came from
	let svelteSvg;
	let x = d3.scaleLinear().rangeRound([0, width]);
	let y = d3.scaleLinear().rangeRound([0, height]);
	let ogRootCount = 0;
	function init() {
		ogRootCount = data.data.node_count;
		// create the svg that we will work with
		svg = d3.select(svelteSvg);
		svg.attr("style", svgStyle).attr("width", width).attr("height", height);
		x.domain([0, width]);
		y.domain([0, height]);

		// pick a color for the rectangles as they descend deeper
		color = treemapColorGenerator(d3.interpolateGreys, data.height, {
			offset: 1,
			reverseColorDirection: false,
		});

		// render the treemap
		group = svg.append("g").call(render, data);
		selectedParent.set(data); // pass to sidebar
	}
	onMount(() => {
		init();
	});

	// used in both zoomin and zoomout
	function copyDims(treemappedNode) {
		return {
			x0: treemappedNode.x0,
			x1: treemappedNode.x1,
			y0: treemappedNode.y0,
			y1: treemappedNode.y1,
		};
	}
	/**
	 * Zooms out `from` a child `to` a parent
	 * @param {{from: HierarchyNode, to: HierarchyNode, fadeDuration?: number, positionDuration?: number}} zoomConfig
	 */
	function zoomout({ from, to } = {}) {
		// select the already rendered group as group0
		const group0 = group.attr("pointer-events", "none");
		// render the new group
		const group1 = (group = svg.insert("g", "*").call(render, to));

		// this is the changes from the render and is now the smaller rectangle inside of the larger
		let fromCopyDims = copyDims(from);

		// rescale to make the outer one much larger to start
		let newX = d3
			.scaleLinear()
			.domain([fromCopyDims.x0, fromCopyDims.x1])
			.rangeRound([0, width]);
		let newY = d3
			.scaleLinear()
			.domain([fromCopyDims.y0, fromCopyDims.y1])
			.rangeRound([0, height]);
		// keep the copy so we can go back
		let prevX = x.copy();
		let prevY = y.copy();

		// this is very reminiscent of change of basis in linear algebra
		// here I update the scaling for positioning
		x = newX;
		y = newY;
		// position the new outer group much larger (what we zoom out to)
		group1.call(position);
		// want to make the larger go smaller, so invert back
		x = x.invert;
		y = y.invert;

		svg.transition()
			.duration(transitionSpeed)
			.call((t) =>
				group0
					.transition(t)
					.remove()
					.attrTween("opacity", () => d3.interpolate(1, 0))
					.call(position, from)
			);
		// revert back to original scaling
		x = prevX;
		y = prevY;
		svg.transition()
			.duration(transitionSpeed)
			.call((t) => group1.transition(t).call(position, to));

		selectedParent.set(to); // pass to sidebar
	}

	/**
	 * Zooms in `to` a child `from` a parent
	 * @param {{from: HierarchyNode, to: HierarchyNode}} zoomConfig
	 */
	function zoomin({ from, to } = {}) {
		let toCopyOfDims = copyDims(to);
		console.log(toCopyOfDims);

		// create the two groups we will transition from
		const group0 = group.attr("pointer-events", "none"); // previous group
		// given the treemap renders in the whole screen, fit in the previous square
		const group1 = (group = svg.append("g").call(render, to)); // new group to zoom into

		// create new scale to size the rectangle smaller inside where it starts
		let newX = d3
			.scaleLinear()
			.domain([0, width])
			.rangeRound([toCopyOfDims.x0, toCopyOfDims.x1]);
		let newY = d3
			.scaleLinear()
			.domain([0, height])
			.rangeRound([toCopyOfDims.y0, toCopyOfDims.y1]);
		let prevX = x.copy();
		let prevY = y.copy();

		x = newX;
		y = newY;
		// group0.selectAll(".leaf").selectAll("image").remove();
		group1.call(position, to);
		x = x.invert;
		y = y.invert;

		svg.transition()
			.duration(transitionSpeed)
			.call((t) => group0.transition(t).remove().call(position, from)); // transition out the old group
		x = prevX;
		y = prevY;
		svg.transition()
			.duration(transitionSpeed)
			.call((t) =>
				group1
					.transition(t)
					.attrTween("opacity", () => d3.interpolate(1, 1))
					.call(position, to)
			); // transition in the new group

		selectedParent.set(to); // pass to sidebar
	}

	/**
	 * this gridifies images within the provided square
	 * @param {number} x0
	 * @param {number} y0
	 * @param {number} x1
	 * @param {number} y1
	 * @param {number} imageWidth
	 * @param {number} imageHeight
	 * @param {any[]} cluster
	 */
	function gridifyImageArray(
		x0,
		y0,
		x1,
		y1,
		imageWidth,
		imageHeight,
		cluster
	) {
		const width = x1 - x0;
		const height = y1 - y0;
		const totalArea = width * height;
		const imageArea = imageWidth * imageHeight;
		const maxNumImages = totalArea / imageArea;

		if (imageArea < 0 || maxNumImages < 0) {
			console.log("negative area");
			return [];
		}

		// next render in a grid
		let flattenedGrid = [];
		const maxNumRows = Math.floor(height / imageHeight);
		const maxNumCols = Math.floor(width / imageWidth);

		// place the images in grid by iterating 2D like over the 1D array
		let currRow = 0;
		let currCol = 0;
		let imagePlaced = 0;
		let currArrayIndex = 0;
		const allImagesFitInGrid = cluster.length <= maxNumImages;

		let currArrayIndexIncrementSize = 1;
		if (!allImagesFitInGrid) {
			const sampleIncrement = Math.floor(cluster.length / maxNumImages);
			currArrayIndexIncrementSize = sampleIncrement;
			if (currArrayIndexIncrementSize === 0)
				currArrayIndexIncrementSize = 1;
		}

		// compare the number of images placed under this method compared to actual
		while (
			currRow < maxNumRows &&
			currArrayIndex < cluster.length &&
			imagePlaced <= maxNumImages
		) {
			let currCluster = cluster[currArrayIndex];
			if (currCluster === undefined) {
				console.log(currCluster, imagePlaced);
				console.log(cluster);
				console.log(
					"total placed:",
					imagePlaced,
					"max:",
					maxNumImages,
					"increment:",
					currArrayIndexIncrementSize
				);
				throw Error();
			}

			// add the grid position directly onto the currCluster object
			// this will copy by reference to the original memory
			currCluster["imagePosition"] = {
				x: currCol * imageWidth,
				y: currRow * imageHeight,
			};

			imagePlaced++; // increment once after each image successfully placed in grid
			currArrayIndex += currArrayIndexIncrementSize; // decides the next item in the array
			flattenedGrid.push(currCluster);

			// increments columns until we need to go to next row
			currCol++;
			if (currCol >= maxNumCols) {
				currCol = 0;
				currRow++;
			}
		}

		return flattenedGrid;
	}

	/**
	 * this function takes a group and renders images on top of it in a grid way
	 * @param {d3.Selection} groupRect
	 * @param {HierarchyNode} d
	 * @param {number} i
	 */
	const renderImages = (groupRect, d, i) => {
		const { x1, x0, y1, y0, cluster } = d;
		let mutatedCluster = cluster;
		// mutatedCluster = mutatedCluster.sort((a, b) => a.index - b.index);
		const imageGrid = gridifyImageArray(
			x0,
			y0,
			x1,
			y1 - topLabelSpace,
			imageWidth,
			imageHeight,
			mutatedCluster
		);

		groupRect
			.selectAll("image")
			.data(imageGrid)
			.join("image")
			.attr("id", (d) => `image-${d.instance_index}`)
			.attr("class", (d) => (d.correct ? "right" : "wrong"))
			.attr("x", (d) => d.imagePosition.x)
			.attr("y", (d) => d.imagePosition.y + topLabelSpace)
			.attr("width", imageWidth)
			.attr("height", imageHeight)
			.attr("href", (d) => `${$imagesEndpoint}/${d.filename}`)
			.attr("cursor", "pointer")
			.on("click", function (event, d) {
				selectedImage.set(d); //pass to sidebar
			})
			.attr("clip-path", d.clip)
			.on("mouseenter", function (event, d) {
				if ($highlightSimilarImages) {
					highlightImages({
						imageGroup: svg.selectAll("image"),
						instancesToHighlight: d.topk_instance_index_list,
						hiddenOpacity,
						highlightedOpacity,
					});
					d3.select(this).attr("opacity", highlightedOpacity);
				}
			})
			.on("mouseleave", function (event, d) {
				if ($highlightSimilarImages) {
					resetOpacity({ highlightedOpacity });
					highlightWrong(
						group,
						$showMisclassifications,
						$changingSizes,
						{
							borderWidth: "1px",
							borderColor: incorrectColor,
						},
						$highlightIncorrectImages
					);
				}
			});
		groupRect
			.selectAll("image")
			.append("title")
			.text(
				(d) =>
					`Click to select image ${d.instance_index}\nactual: ${d.true_class}, pred: ${d.predicted_class}`
			);
	};

	/**
	 * Only job is to position the nodes in group where they need to go in the treemap
	 * @param {d3.Selection} group
	 * @param {HierarchyNode} root
	 */
	function position(group, root) {
		group.attr("transform", "translate(0, 10)"); // quick fix that probably covers up an underlying problem
		const subGroups = group.selectChildren("g");
		subGroups
			.attr("transform", (d) => {
				if (d.x0 === undefined || d.y0 === undefined) {
					console.log(d);
					throw Error("These must be present to render");
				}
				return `translate(${x(d.x0) - innerPadding / 2},${
					y(d.y0) - innerPadding / 2
				})`;
			})
			.select(".treemap-rect")
			.attr("width", (d) => x(d.x1) - x(d.x0) + innerPadding)
			.attr(
				"height",
				(d) => y(d.y1) - y(d.y0) + innerPadding + topPadding
			);
		subGroups
			.select(".label-text")
			.attr("x", 5)
			.attr("y", 13)
			.attr("pointer-events", "none");
	}

	/**
	 * renders the treemap to the desired group <g></g>
	 * @param {d3.Selection} group
	 * @param {HierarchyNode} root
	 */
	function render(group, root) {
		// call the custom treemap function which returns the nodes to render in an array
		/** @type {Node[]}*/
		const nodesToRender = kClustersTreeMap({
			parent: root,
			x0: 0,
			y0: 0,
			x1: width,
			y1: height,
			kClusters: numClusters,
			imageWidth,
			imageHeight,
		});

		// renders the groups and labels the leaf nodes with class .leaf
		const node = group
			.selectAll("g")
			.data(nodesToRender)
			.join("g")
			.attr("id", (d) => `g-${d.data.node_index}`)
			.attr("class", (d) => (d.isLeaf ? "leaf" : ""));

		node.append("rect").call(renderTreemapRect, root);
		// I don't want the text to extend past the rectangle, so clip it off
		node.append("clipPath")
			.attr("id", (d) => {
				d.clip = new ID(`clip-${d.data.node_index}`);
				return d.clip.id;
			})
			.append("use")
			.attr("xlink:href", (d) => d.rect.href);
		node.append("text").call(
			renderLabel,
			!$hideLabelAccuracy,
			!$hideLabelCoverage
		);
		// for each <g> that shows a cluster on top, render the images on top
		const currentLeaves = group.selectAll(".leaf"); // local leaves basically
		forEachSelection(currentLeaves, renderImages);

		group.call(position, root);
	}

	function renderTreemapRect(rect, root) {
		const colorByRemainingHeight = (d, i) =>
			(d.rectColor = d3.color(color(d.height))); // function to color the rectangles
		// now create the rectangle

		rect.attr("id", (d) => {
			// this is neccesary to clip/crop the things that extend out of the rectangle (potentially the text)
			// clip-path shows up in a few lines and takes d.rect.href
			d.rect = new ID(`rect-${d.data.node_index}`); // adds id and href property
			return d.rect.id;
		})
			.attr("fill", colorByRemainingHeight) //appends rectColor to d
			.attr("stroke", (d, i) => {
				if (false && i === 0 && d !== data) {
					d.strokeColor = d3.color("steelblue").brighter(0.2);
				} else {
					d.strokeColor = d.rectColor.darker(0.2);
				}
				return d.strokeColor;
			})
			.attr("stroke-width", 1.5)
			.attr("class", "treemap-rect")
			.on("click", function (event, child) {
				const clickedIsLeaf = child.data.leaf;
				// leaves must not be clicked
				if (!clickedIsLeaf) {
					// desired behavior: zoom out when root node is clicked
					const clickedIsRootNode = child === root;
					if (clickedIsRootNode) {
						const prevRoot = prevRoots.pop(); // stack to keep track the roots we click on
						if (prevRoot !== undefined) {
							zoomout({
								from: root,
								to: prevRoot,
							});
						}
					} else {
						function removeRectsAbove(child) {
							// removes nodes that are rendered after the one we clicked on
							// this is done because the animation breaks down for these rectangles
							function removeRenderedHierarchy(parent) {
								let nodeIds = [];
								function _removeRenderedHierarchy(_parent) {
									if (
										_parent.isLeaf ||
										_parent.children === undefined
									)
										return;
									_parent.children.forEach((child) => {
										nodeIds.push(child.data.node_index);
										_removeRenderedHierarchy(child);
									});
								}
								_removeRenderedHierarchy(parent);
								return nodeIds;
							}
							// remove the nodes above and the current node aswell as all the text
							const nodesToRemove =
								removeRenderedHierarchy(child);
							nodesToRemove.forEach((node_index) =>
								d3.select(`#g-${node_index}`).remove()
							);
							d3.select(`#g-${child.data.node_index}`).remove();
						}

						removeRectsAbove(child); //removes rects above the current child
						// node.select("text").remove(); // removes all text labels

						// adds to the stack to show we traversed down here
						// necessary to zoomout
						prevRoots.push(root);
						// then zoomin
						zoomin({
							to: child,
							from: root,
						});
					}
				}
			})
			.on("mouseover", function (event, child) {
				if (child !== root && !child.data.leaf) {
					d3.select(this).attr("stroke", "darkgrey");
				}
				nodeHovering.set(child);
			})
			.on("mouseout", function (event, child) {
				if (child !== root) {
					d3.select(this)
						.attr("stroke", child.strokeColor)
						.attr("fill", child.rectColor);
				}
				nodeHovering.set(null); // this can be omitted
			});
		rect.append("title").text((d, i) => {
			let clusterLabel = ``;
			if (d === root && d !== data) {
				clusterLabel = `Click to go back from cluster`;
			} else if (d === data) {
			} else if (d.data.leaf) {
				clusterLabel = `Only 1 image, can't go further`;
			} else {
				clusterLabel = `Click to go into cluster`;
			}
			return clusterLabel;
		});
		rect.attr("cursor", (d) =>
			d.data.leaf || d === data ? "default" : "pointer"
		);
	}
	function renderLabel(text, showAccuracy, showCoverage) {
		text.attr("clip-path", (d) => d.clip)
			.text((d) => {
				let totalLabel = `${d.data.node_count} image${
					d.data.node_count > 1 ? "s" : ""
				}`;
				if (showAccuracy) {
					let accuracy = d.data.accuracy;
					if (accuracy === undefined) {
						const leafCorrect =
							d.data.predicted_class === d.data.true_class;
						accuracy = leafCorrect ? 1.0 : 0.0;
					}
					let accuracyLabel = `${toPercent(accuracy)} accuracy`;
					totalLabel += `, ${accuracyLabel}`;
				}
				if (showCoverage) {
					let incorrectCount =
						d.data.node_count - d.data.correct_count;
					let coverage = incorrectCount / ogRootCount;
					if (coverage === undefined) {
						const leafCorrect =
							d.data.predicted_class === d.data.true_class;
						coverage = leafCorrect ? 0.0 : 1.0 / ogRootCount;
					}
					let coverageLabel = `${toPercent(coverage)} coverage`;
					totalLabel += `, ${coverageLabel}`;
				}
				return totalLabel;
			})
			.attr("fill", (d) => (d.height < 5 ? "white" : "black"))
			.style("font-size", "10px")
			.attr("class", "label-text");
	}

	// this is absolutely the worst way to do it, fix this later

	function resetWithNewSettings(svg, numClusters, imageSize) {
		svg.selectAll("g").remove();
		init();
	}
	// when the number of clusters is changed in the sidebar or image size, reset the treemap with those dimensions
	$: {
		if (svelteSvg && $changingSizes) {
			// let svg = d3.select(svelteSvg);
			// // resetWithNewSettings(svg, $treemapNumClusters, $treemapImageSize);
			// make it so this is not called otherwise
			if ($treemapNumClusters && $treemapImageSize) {
				group.selectChildren("g").remove();
				render(group, $selectedParent);
			}
		}
	}
	function highlightWrong(
		group,
		showMisclassifications,
		changingSizes,
		{ borderWidth = "1px", borderColor = incorrectColor },
		focus
	) {
		function _showingStroke(d, color, width) {
			d.style = `${width} solid ${color}`;
			return d.style;
		}
		function _showingOpacity(d) {
			d.opacity = highlightedOpacity;
			return d.opacity;
		}
		function _hiddenOpacity(d) {
			d.opacity = hiddenOpacity;
			return d.opacity;
		}
		if (group) {
			const wrongImages = group.selectAll(".wrong");
			const rightImages = group.selectAll(".right");
			wrongImages.attr("opacity", _showingOpacity);
			rightImages.attr("opacity", _showingOpacity);
			if (showMisclassifications && group) {
				if (focus) {
					rightImages.attr("opacity", _hiddenOpacity);
				}
				wrongImages.style("outline", (d) =>
					_showingStroke(d, borderColor, borderWidth)
				);
				wrongImages.raise(); // bring to top of rendering
			} else {
				if (focus) {
					rightImages.attr("opacity", _hiddenOpacity);
				}
				wrongImages.style("outline", (d) =>
					_showingStroke(d, "transparent", borderWidth)
				);
			}
			group
				.selectAll("text")
				.call(renderLabel, !$hideLabelAccuracy, !$hideLabelCoverage);
		}
	}
	$: {
		if (!$hideMisclassifiedImages) {
			highlightWrong(
				group,
				$showMisclassifications,
				$changingSizes,
				{
					borderWidth: "1px",
					borderColor: incorrectColor,
				},
				$highlightIncorrectImages
			);
		}
	}
</script>

<svg bind:this={svelteSvg} viewbox="0 0 {width} {height + 30}" />

<style>
	svg {
		/* border: 1px solid gray; */
		overflow: hidden;
	}
</style>
