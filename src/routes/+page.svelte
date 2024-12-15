<script lang="ts">
	import { onMount } from 'svelte';
	import Button from '$lib/components/ui/button/button.svelte';
	import * as Dialog from '$lib/components/ui/dialog';
	import { Input } from '$lib/components/ui/input';
	import { recipe } from '$lib/stores/recipeStore.svelte';
	import Sortable from 'sortablejs';

	//instructions management
	let newInstruction = $state('');
	let instructionsList: HTMLElement;
	let dialogOpen = $state(false);
	let key = $state(0); // Force re-render key

	// Ingredients management
	let newIngredient = $state('');
	let ingredientsList: HTMLElement;
	let editingIngredientIndex = $state<number | null>(null);
	let editingIngredientText = $state('');

	// Editing state
	let editingIndex = $state<number | null>(null);
	let editingText = $state('');

	// Load from URL on mount
	onMount(() => {
		const params = new URLSearchParams(window.location.search);
		recipe.name = params.get('name') || '';
		recipe.instructions = JSON.parse(params.get('instructions') || '[]');
		recipe.ingredients = JSON.parse(params.get('ingredients') || '[]');
	});

	// Sync to URL when recipe changes
	$effect(() => {
		const params = new URLSearchParams();
		params.set('name', recipe.name);
		params.set('instructions', JSON.stringify(recipe.instructions));
		params.set('ingredients', JSON.stringify(recipe.ingredients));
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

	// Handle ingredients sorting
	$effect(() => {
		if (dialogOpen && ingredientsList) {
			const sortable = new Sortable(ingredientsList, {
				animation: 150,
				handle: '.handle',
				group: 'ingredients',
				draggable: 'li',
				onEnd: (evt) => {
					const items = [...recipe.ingredients];
					const [removed] = items.splice(evt.oldIndex!, 1);
					items.splice(evt.newIndex!, 0, removed);
					recipe.ingredients = items;
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

	function addIngredient() {
		if (newIngredient.trim()) {
			recipe.ingredients = [...recipe.ingredients, newIngredient.trim()];
			newIngredient = '';
		}
	}
	function truncateWords(text: string, wordCount: number = 10): string {
		const words = text.split(' ');
		if (words.length <= wordCount) return text;
		return words.slice(0, wordCount).join(' ') + '...';
	}

	function startEditInstruction(index: number, text: string) {
		editingIndex = index;
		editingText = text;
	}
	function saveEditInstruction() {
		if (editingIndex !== null) {
			const newInstructions = [...recipe.instructions];
			newInstructions[editingIndex] = editingText;
			recipe.instructions = newInstructions;
			editingIndex = null;
		}
	}

	function startEditIngredient(index: number, text: string) {
		editingIngredientIndex = index;
		editingIngredientText = text;
	}

	function saveEditIngredient() {
		if (editingIngredientIndex !== null) {
			const newIngredients = [...recipe.ingredients];
			newIngredients[editingIngredientIndex] = editingIngredientText;
			recipe.ingredients = newIngredients;
			editingIngredientIndex = null;
		}
	}
</script>

<Dialog.Root onOpenChange={(open) => (dialogOpen = open)}>
	<Dialog.Trigger>
		<Button class="m-2 rounded-none border border-white">Edit Recipe</Button>
	</Dialog.Trigger>
	<Dialog.Content class="max-h-screen w-full min-w-full max-w-full overflow-y-auto">
		<Dialog.Header class="bg-black/2 mx-auto flex w-full items-center justify-between">
			<Dialog.Title class="mb-4">Recipe</Dialog.Title>
			<Dialog.Description class="lg:w-1/2">
				<Input type="text" placeholder="Recipe name" bind:value={recipe.name} class="mb-8" />
				<div class="items-start justify-start gap-4 *:w-full lg:flex">
					<div>
						<div class="flex gap-2">
							<Input type="text" placeholder="Add ingredient" bind:value={newIngredient} />
							<Button onclick={addIngredient}>Add</Button>
						</div>
						<ol bind:this={ingredientsList} class="list-decimal space-y-2">
							{#each recipe.ingredients as ingredient, i (ingredient + i)}
								<li
									class="flex select-none items-center justify-between rounded-md bg-secondary/20 px-2 text-left"
								>
									<div class="flex flex-1 items-start gap-1">
										<span class="handle cursor-move select-none">⋮⋮</span>
										{#if editingIngredientIndex === i}
											<Input
												type="text"
												bind:value={editingIngredientText}
												class="flex-1"
												onblur={saveEditIngredient}
												onkeydown={(e) => e.key === 'Enter' && saveEditIngredient()}
											/>
										{:else}
											<span class="select-none">{truncateWords(ingredient)}</span>
										{/if}
									</div>
									<div class="flex gap-1">
										<Button
											variant="ghost"
											size="sm"
											onclick={() => startEditIngredient(i, ingredient)}
										>
											✎
										</Button>
										<Button
											variant="ghost"
											size="sm"
											onclick={() => {
												recipe.ingredients = recipe.ingredients.filter((_, index) => index !== i);
											}}
										>
											×
										</Button>
									</div>
								</li>
							{/each}
						</ol>
					</div>
					<div class="mt-8 lg:mt-0">
						<div class="flex gap-2">
							<Input type="text" placeholder="Add instruction" bind:value={newInstruction} />
							<Button onclick={addInstruction}>Add</Button>
						</div>
						<ol bind:this={instructionsList} class="list-decimal space-y-2">
							{#each recipe.instructions as instruction, i (instruction + i + key)}
								<li
									class="flex select-none items-center justify-between rounded-md bg-secondary/20 px-2 text-left"
								>
									<div class="flex flex-1 items-start gap-1">
										<span class="handle cursor-move select-none">⋮⋮</span>
										{#if editingIndex === i}
											<Input
												type="text"
												bind:value={editingText}
												class="flex-1"
												onblur={saveEditInstruction}
												onkeydown={(e) => e.key === 'Enter' && saveEditInstruction()}
											/>
										{:else}
											<span class="select-none">{truncateWords(instruction)}</span>
										{/if}
									</div>
									<div class="flex gap-1">
										<Button
											variant="ghost"
											size="sm"
											onclick={() => startEditInstruction(i, instruction)}
										>
											✎
										</Button>
										<Button
											variant="ghost"
											size="sm"
											onclick={() => {
												recipe.instructions = recipe.instructions.filter((_, index) => index !== i);
											}}
										>
											×
										</Button>
									</div>
								</li>
							{/each}
						</ol>
					</div>
				</div>
			</Dialog.Description>
		</Dialog.Header>
	</Dialog.Content>
</Dialog.Root>

<div class="mx-auto max-w-2xl space-y-4 rounded-xl bg-white/10 p-4 text-white lg:p-12">
	<h1 class="text-2xl font-bold">{recipe.name}</h1>

	{#if recipe.ingredients.length}
		<div class="prose">
			<h2>Ingredients</h2>
			<ul class="list-disc pl-4">
				{#each recipe.ingredients as ingredient}
					<li>{ingredient}</li>
				{/each}
			</ul>
		</div>
	{/if}

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
