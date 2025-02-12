<script lang="ts">
	import Toggle from '$lib/components/Toggle.svelte'
	import { getContext } from 'svelte'
	import type { AppInput } from '../../inputType'
	import type { Output } from '../../rx'
	import type { AppViewerContext, ComponentCustomCSS } from '../../types'
	import { concatCustomCss } from '../../utils'
	import AlignWrapper from '../helpers/AlignWrapper.svelte'
	import InputValue from '../helpers/InputValue.svelte'

	export let id: string
	export let configuration: Record<string, AppInput>
	export let horizontalAlignment: 'left' | 'center' | 'right' | undefined = undefined
	export let verticalAlignment: 'top' | 'center' | 'bottom' | undefined = undefined
	export let customCss: ComponentCustomCSS<'text'> | undefined = undefined
	export let render: boolean

	const { app, worldStore } = getContext<AppViewerContext>('AppViewerContext')

	export const staticOutputs: string[] = ['result']

	let defaultValue: boolean | undefined = undefined
	let labelValue: string = 'Default label'

	// As the checkbox is a special case and has no input
	// we need to manually set the output

	$: outputs = $worldStore?.outputsById[id] as {
		result: Output<boolean>
	}

	$: defaultValue != undefined && outputs?.result.set(defaultValue)

	$: css = concatCustomCss($app.css?.checkboxcomponent, customCss)
</script>

<InputValue {id} input={configuration.label} bind:value={labelValue} />
<InputValue {id} input={configuration.defaultValue} bind:value={defaultValue} />

<AlignWrapper {render} {horizontalAlignment} {verticalAlignment}>
	<Toggle
		on:pointerdown={(e) => {
			e?.stopPropagation()
			window.dispatchEvent(new Event('pointerup'))
		}}
		checked={defaultValue}
		options={{ right: labelValue }}
		textClass={css?.text?.class ?? ''}
		textStyle={css?.text?.style ?? ''}
		on:change={(e) => {
			outputs.result.set(e.detail)
		}}
	/>
</AlignWrapper>
