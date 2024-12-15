<script lang="ts">
	import { onMount } from 'svelte';
	import Button from '$lib/components/ui/button/button.svelte';
	import * as Dialog from '$lib/components/ui/dialog';
	import { Input } from '$lib/components/ui/input';
	import { recipe } from '$lib/stores/recipeStore.svelte';
	import Sortable from 'sortablejs';

	let newInstruction = $state('');
	let instructionsList: HTMLElement;
	let dialogOpen = $state(false);
	let key = $state(0); // Force re-render key

	// Load from URL on mount
	onMount(() => {
		const params = new URLSearchParams(window.location.search);
		recipe.name = params.get('name') || '';
		recipe.instructions = JSON.parse(params.get('instructions') || '[]');
	});

	// Sync to URL when recipe changes
	$effect(() => {
		const params = new URLSearchParams();
		params.set('name', recipe.name);
		params.set('instructions', JSON.stringify(recipe.instructions));
		window.history.replaceState({}, '', `?${params.toString()}`);
	});

	// Handle instructions sorting
	$effect(() => {
		if (dialogOpen && instructionsList) {
			const sortable = new Sortable(instructionsList, {
				animation: 150,
				handle: '.handle',
				group: 'instructions',
				draggable: 'li',
				onEnd: (evt) => {
					const newArray = [...recipe.instructions];
					const [removed] = newArray.splice(evt.oldIndex!, 1);
					newArray.splice(evt.newIndex!, 0, removed);
					recipe.instructions = newArray;
					key++; // Force re-render of list
				}
			});

			return () => sortable.destroy();
		}
	});

	function addInstruction() {
		if (newInstruction.trim()) {
			recipe.instructions = [...recipe.instructions, newInstruction.trim()];
			newInstruction = '';
		}
	}
</script>

<Dialog.Root onOpenChange={(open) => (dialogOpen = open)}>
	<Dialog.Trigger>
		<Button variant="outline">Edit Recipe</Button>
	</Dialog.Trigger>
	<Dialog.Content>
		<Dialog.Header>
			<Dialog.Title class="mb-4">Recipe</Dialog.Title>
			<Dialog.Description>
				<div class="space-y-4">
					<Input type="text" placeholder="Recipe name" bind:value={recipe.name} />
					<div class="flex gap-2">
						<Input type="text" placeholder="Add instruction step" bind:value={newInstruction} />
						<Button onclick={addInstruction}>Add</Button>
					</div>
					<ol bind:this={instructionsList} class="list-decimal space-y-2 pl-4">
						{#each recipe.instructions as instruction, i (instruction + i + key)}
							<li
								class="flex select-none items-center justify-between rounded-md bg-secondary/20 p-2 pl-2"
							>
								<div class="flex items-center gap-2">
									<span class="handle cursor-move select-none">⋮⋮</span>
									<span class="select-none">{instruction}</span>
								</div>
								<Button
									variant="ghost"
									size="sm"
									onclick={() => {
										recipe.instructions = recipe.instructions.filter((_, index) => index !== i);
									}}
								>
									×
								</Button>
							</li>
						{/each}
					</ol>
				</div>
			</Dialog.Description>
		</Dialog.Header>
	</Dialog.Content>
</Dialog.Root>

<div class="mx-auto max-w-2xl space-y-4 p-4">
	<h1 class="text-2xl font-bold">{recipe.name}</h1>
	{#if recipe.instructions.length}
		<div class="prose">
			<h2>Instructions</h2>
			<ol class="list-decimal pl-4">
				{#each recipe.instructions as instruction}
					<li>{instruction}</li>
				{/each}
			</ol>
		</div>
	{/if}
</div>
