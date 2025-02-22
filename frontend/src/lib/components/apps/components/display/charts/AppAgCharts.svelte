<script lang="ts">
	import RunnableWrapper from '../../helpers/RunnableWrapper.svelte'
	import type { AppInput } from '../../../inputType'
	import type {
		AppViewerContext,
		ComponentCustomCSS,
		RichConfiguration,
		RichConfigurations
	} from '../../../types'
	import { initCss } from '../../../utils'
	import { getContext, onMount } from 'svelte'
	import { initConfig, initOutput } from '../../../editor/appUtils'
	import { components } from '../../../editor/component'
	import ResolveConfig from '../../helpers/ResolveConfig.svelte'
	import { twMerge } from 'tailwind-merge'
	import ResolveStyle from '../../helpers/ResolveStyle.svelte'
	import type { AgChartOptions, AgChartInstance } from 'ag-charts-community'

	export let id: string
	export let componentInput: AppInput | undefined
	export let configuration: RichConfigurations
	export let initializing: boolean | undefined = undefined
	export let customCss: ComponentCustomCSS<'agchartscomponent'> | undefined = undefined
	export let render: boolean
	export let datasets: RichConfiguration | undefined
	export let xData: RichConfiguration | undefined
	export let license: string | undefined = undefined
	export let ee: boolean = false

	const { app, worldStore } = getContext<AppViewerContext>('AppViewerContext')

	type Dataset = {
		value: RichConfiguration
		name: string
		type: 'bar' | 'line' | 'scatter' | 'area' | 'range-bar'
	}

	let resolvedDatasets: Dataset[]
	let resolvedXData: number[] = []

	const outputs = initOutput($worldStore, id, {
		result: undefined as
			| {
					data: any[]
					series: any[]
			  }
			| undefined,
		loading: false
	})

	let result: undefined = undefined

	const resolvedConfig = initConfig(
		components['agchartscomponent'].initialData.configuration,
		configuration
	)

	let css = initCss($app.css?.agchartscomponent, customCss)
	let chartInstance: AgChartInstance | undefined = undefined

	function updateChart() {
		if (!chartInstance) {
			return
		}

		let data = [] as any[]

		for (let i = 0; i < resolvedXData.length; i++) {
			const o = {
				x: resolvedXData[i]
			}

			for (let j = 0; j < resolvedDatasets.length; j++) {
				if (!resolvedDatasetsValues[j]) {
					continue
				}

				if (resolvedDatasetsValues[j].type === 'range-bar') {
					o[`y-${j}-low`] = resolvedDatasetsValues[j].value[i]?.[0]
					o[`y-${j}-high`] = resolvedDatasetsValues[j].value[i]?.[1]
				} else {
					o[`y-${j}`] = resolvedDatasetsValues[j].value[i]
				}
			}

			data.push(o)
		}

		const options = {
			container: document.getElementById(`agchart-${id}`) as HTMLElement,
			data: data,
			series:
				(resolvedDatasets?.map((d, index) => {
					const type = resolvedDatasetsValues[index].type
					if (type === 'range-bar') {
						return {
							type: type,
							xKey: 'x',
							yLowKey: `y-${index}-low`,
							yHighKey: `y-${index}-high`,
							yName: d.name
						}
					} else {
						return {
							type: type,
							xKey: 'x',
							yKey: `y-${index}`,
							yName: d.name
						}
					}
				}) as any[]) ?? []
		}

		outputs.result.set({
			data: options.data,
			series: options.series
		})

		AgChartsInstance?.update(chartInstance, options)
	}

	$: resolvedDatasetsValues = resolvedDatasets?.map((d) => {
		const config = initConfig(
			{
				value: {
					type: 'oneOf',
					selected: 'bar',
					labels: {
						bar: 'Bar',
						scatter: 'Scatter',
						line: 'Line',
						area: 'Area',
						['range-bar']: 'Range Bar'
					},
					configuration: {
						bar: {
							value: {
								type: 'static',
								fieldType: 'array',
								subFieldType: 'number',
								value: [25, 25, 50]
							}
						},
						scatter: {
							value: {
								type: 'static',
								fieldType: 'array',
								subFieldType: 'number',
								value: [25, 25, 50]
							}
						},
						line: {
							value: {
								type: 'static',
								fieldType: 'array',
								subFieldType: 'number',
								value: [25, 25, 50]
							}
						},
						area: {
							value: {
								type: 'static',
								fieldType: 'array',
								subFieldType: 'number',
								value: [25, 25, 50]
							}
						},
						['range-bar']: {
							value: {
								type: 'static',
								fieldType: 'array',
								subFieldType: 'number-tuple',
								value: [
									[10, 15],
									[20, 25],
									[18, 27]
								]
							}
						}
					}
				}
			},
			{
				value: d.value
			}
		)

		return {
			type: d.value['selected'],
			// @ts-ignore
			value: config.value?.['configuration']?.[d.value['selected']].value
		}
	})

	$: resolvedXData && resolvedDatasets && resolvedDatasetsValues && chartInstance && updateChart()
	$: result && updateChartByResult()

	function updateChartByResult() {
		if (!result || !chartInstance) {
			return
		}

		const options = {
			container: document.getElementById(`agchart-${id}`) as HTMLElement,
			data: result?.['data'],
			series: result?.['series']
		}

		outputs.result.set({
			data: result?.['data'],
			series: result?.['series']
		})

		AgChartsInstance?.update(chartInstance, options)
	}

	let AgChartsInstance: any | undefined = undefined

	async function loadLibrary() {
		if (ee) {
			const enterprise = await import('ag-charts-enterprise')
			AgChartsInstance = enterprise.AgCharts

			AgChartsInstance.setLicenseKey(license)
		} else {
			const community = await import('ag-charts-community')
			AgChartsInstance = community.AgCharts
		}
	}

	onMount(() => {
		loadLibrary().then(() => {
			try {
				// Chart Options
				const options: AgChartOptions = {
					container: document.getElementById(`agchart-${id}`) as HTMLElement,
					data: [],
					series: []
				}

				chartInstance = AgChartsInstance?.create(options)
			} catch (error) {
				console.error(error)
			}
		})
	})
</script>

{#if datasets}
	<ResolveConfig
		{id}
		key={'datasets'}
		bind:resolvedConfig={resolvedDatasets}
		configuration={datasets}
	/>
{/if}

{#if xData}
	<ResolveConfig {id} key={'xData'} bind:resolvedConfig={resolvedXData} configuration={xData} />
{/if}

{#if resolvedDatasets}
	{#each resolvedDatasets as resolvedDataset, index (resolvedDataset.name + index)}
		<ResolveConfig
			{id}
			key={'datasets' + index}
			extraKey={resolvedDataset.name}
			bind:resolvedConfig={resolvedDatasetsValues[index]}
			configuration={resolvedDataset.value}
		/>
	{/each}
{/if}

{#each Object.keys(components['agchartscomponent'].initialData.configuration) as key (key)}
	<ResolveConfig
		{id}
		{key}
		bind:resolvedConfig={resolvedConfig[key]}
		configuration={configuration[key]}
	/>
{/each}

{#each Object.keys(css ?? {}) as key (key)}
	<ResolveStyle
		{id}
		{customCss}
		{key}
		bind:css={css[key]}
		componentStyle={$app.css?.chartjscomponent}
	/>
{/each}

<RunnableWrapper {outputs} {render} autoRefresh {componentInput} {id} bind:initializing bind:result>
	<div
		class={twMerge('w-full h-full', css?.container?.class, 'wm-agchart')}
		style={css?.container?.style ?? ''}
	>
		<div id={`agchart-${id}`} class="h-full w-full" />
	</div>
</RunnableWrapper>
