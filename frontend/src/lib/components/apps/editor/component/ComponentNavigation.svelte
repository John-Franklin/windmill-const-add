<script lang="ts">
	import { getContext } from 'svelte'
	import {
		findGridItem,
		findGridItemParentGrid,
		getAllSubgridsAndComponentIds,
		insertNewGridItem,
		sortGridItemsPosition
	} from '../appUtils'
	import type { AppEditorContext, AppViewerContext, EditorBreakpoint, GridItem } from '../../types'
	import { push } from '$lib/history'
	import { sendUserToast } from '$lib/utils'

	const { app, selectedComponent, breakpoint, focusedGrid, componentControl } =
		getContext<AppViewerContext>('AppViewerContext')

	const { history } = getContext<AppEditorContext>('AppEditorContext')

	let tempGridItem: GridItem | undefined = undefined
	let copiedGridItem: GridItem | undefined = undefined

	function getSortedGridItemsOfChildren(): GridItem[] {
		if (!$focusedGrid) {
			return sortGridItemsPosition($app.grid, $breakpoint)
		}

		if (!$app.subgrids) {
			return []
		}

		return sortGridItemsPosition(
			$app.subgrids[`${$focusedGrid.parentComponentId}-${$focusedGrid.subGridIndex}`],
			$breakpoint
		)
	}

	function getSortedGridItemsOfGrid(): GridItem[] {
		if ($app.grid.find((item) => item.id === $selectedComponent)) {
			return sortGridItemsPosition($app.grid, $breakpoint)
		}

		if (!$app.subgrids) {
			return []
		}

		return sortGridItemsPosition(
			Object.values($app.subgrids ?? {}).find((grid) =>
				grid.find((item) => item.id === $selectedComponent)
			) ?? [],
			$breakpoint
		)
	}

	function left(event: KeyboardEvent) {
		if (!$componentControl[$selectedComponent!]?.left?.()) {
			const sortedGridItems = getSortedGridItemsOfGrid()
			const currentIndex = sortedGridItems.findIndex((item) => item.id === $selectedComponent)

			if (currentIndex !== -1 && currentIndex > 0) {
				$selectedComponent = sortedGridItems[currentIndex - 1].id
			}
		}

		event.preventDefault()
	}

	function right(event: KeyboardEvent) {
		let r = $componentControl[$selectedComponent!]?.right?.()
		if (!r) {
			const sortedGridItems = getSortedGridItemsOfGrid()
			const currentIndex = sortedGridItems.findIndex((item) => item.id === $selectedComponent)

			if (currentIndex !== -1 && currentIndex < sortedGridItems.length - 1) {
				$selectedComponent = sortedGridItems[currentIndex + 1].id
			}
		}

		event.preventDefault()
	}

	function down(event: KeyboardEvent) {
		if (!$focusedGrid) {
			$selectedComponent = getSortedGridItemsOfChildren()[0]?.id
			event.preventDefault()
		} else if ($app.subgrids) {
			const index = $focusedGrid?.subGridIndex ?? 0
			const subgrid = $app.subgrids[`${$selectedComponent}-${index}`]

			if (!subgrid || subgrid.length === 0) {
				return
			}

			const sortedGridItems = sortGridItemsPosition(subgrid, $breakpoint)

			if (sortedGridItems) {
				$selectedComponent = sortedGridItems[0].id
			}
			event.preventDefault()
		}
	}

	function handleEscape(event: KeyboardEvent) {
		$selectedComponent = undefined
		$focusedGrid = undefined
		event.preventDefault()
	}

	function handleArrowUp(event: KeyboardEvent) {
		if (!$selectedComponent) return
		let parentId = findGridItemParentGrid($app, $selectedComponent)?.split('-')[0]

		if (parentId) {
			$selectedComponent = parentId
		} else {
			$selectedComponent = undefined
			$focusedGrid = undefined
		}
	}

	function handleCopy(event: KeyboardEvent) {
		if (!$selectedComponent) {
			return
		}
		tempGridItem = undefined
		copiedGridItem = findGridItem($app, $selectedComponent)
	}

	function handleCut(event: KeyboardEvent) {
		if (!$selectedComponent) {
			return
		}
		push(history, $app)

		const gridItem = findGridItem($app, $selectedComponent)
		copiedGridItem = gridItem

		if (!gridItem) {
			return
		}

		// Store the grid item in a temp variable so we can paste it later
		tempGridItem = gridItem
	}

	function handlePaste(event: KeyboardEvent) {
		push(history, $app)

		if (tempGridItem) {
			if (
				$focusedGrid &&
				getAllSubgridsAndComponentIds($app, tempGridItem.data)[0].includes(
					`${$focusedGrid.parentComponentId}-${$focusedGrid.subGridIndex}`
				)
			) {
				sendUserToast('Cannot paste a component into itself', true)
				return
			}
			let parentGrid = findGridItemParentGrid($app, tempGridItem.data.id)
			const grid = parentGrid ? $app.subgrids![parentGrid] : $app.grid
			let idx = grid.findIndex((item) => {
				return item.id == tempGridItem!.data.id
			})
			if (idx > -1) {
				grid.splice(idx, 1)
			}

			insertNewGridItem($app, tempGridItem.data, $focusedGrid, true)
			copiedGridItem = tempGridItem
			tempGridItem = undefined
		} else if (copiedGridItem) {
			insertNewGridItem($app, copiedGridItem.data, $focusedGrid, false)
		}

		$app = $app
	}

	function keydown(event: KeyboardEvent) {
		switch (event.key) {
			case 'Escape':
				handleEscape(event)
				break

			case 'ArrowUp': {
				handleArrowUp(event)
				break
			}

			case 'ArrowDown': {
				down(event)
				break
			}

			case 'ArrowRight': {
				right(event)
				break
			}

			case 'ArrowLeft': {
				left(event)
				break
			}

			case 'c':
				if (event.ctrlKey || event.metaKey) {
					handleCopy(event)
				}
				break

			case 'v':
				if (event.ctrlKey || event.metaKey) {
					handlePaste(event)
				}
				break

			case 'x':
				if (event.ctrlKey || event.metaKey) {
					handleCut(event)
				}
				break

			default:
				break
		}
	}
</script>

<svelte:window on:keydown={keydown} />
