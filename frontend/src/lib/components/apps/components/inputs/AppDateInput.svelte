<script lang="ts">
	import { getContext } from 'svelte'
	import { twMerge } from 'tailwind-merge'
	import type { AppInput } from '../../inputType'
	import type { Output } from '../../rx'
	import type { AppViewerContext, ComponentCustomCSS } from '../../types'
	import { concatCustomCss } from '../../utils'
	import AlignWrapper from '../helpers/AlignWrapper.svelte'
	import InputValue from '../helpers/InputValue.svelte'

	export let id: string
	export let configuration: Record<string, AppInput>
	export let inputType: 'date'
	export let verticalAlignment: 'top' | 'center' | 'bottom' | undefined = undefined
	export const staticOutputs: string[] = ['result']
	export let customCss: ComponentCustomCSS<'input'> | undefined = undefined
	export let render: boolean

	const { app, worldStore, selectedComponent } = getContext<AppViewerContext>('AppViewerContext')
	let labelValue: string = 'Title'
	let minValue: string = ''
	let maxValue: string = ''
	let defaultValue: string | undefined = undefined

	let value: string | undefined = undefined

	$: outputs = $worldStore?.outputsById[id] as {
		result: Output<string | undefined>
	}

	$: handleDefault(defaultValue)

	$: outputs?.result.set(value)

	function handleDefault(defaultValue: string | undefined) {
		value = defaultValue
	}
	$: css = concatCustomCss($app.css?.dateinputcomponent, customCss)
</script>

<InputValue {id} input={configuration.label} bind:value={labelValue} />
<InputValue {id} input={configuration.minDate} bind:value={minValue} />
<InputValue {id} input={configuration.maxDate} bind:value={maxValue} />
<InputValue {id} input={configuration.defaultValue} bind:value={defaultValue} />

<AlignWrapper {render} {verticalAlignment}>
	{#if inputType === 'date'}
		<input
			on:focus={() => ($selectedComponent = id)}
			on:pointerdown|stopPropagation
			type="date"
			bind:value
			min={minValue}
			max={maxValue}
			placeholder="Type..."
			class={twMerge('mx-0.5', css?.input?.class ?? '')}
			style={css?.input?.style ?? ''}
		/>
	{/if}
</AlignWrapper>
