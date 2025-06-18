<script lang="ts">
	import {
		addTodo,
		deleteTodo,
		getTodos,
		toggleTodo,
		updateTodo,
		toggleEditing
	} from './todos.remote'

	// this behaves like a regular function but uses RPC
	const todos = getTodos()
</script>

<main>
	<h1>Todo App</h1>

	<!-- using enhance to customize how the form is progressively enhanced -->
	<form
		{...addTodo.enhance(async ({ form, submit, data }) => {
			// get form data
			const text = data.get('text')!.toString().trim()

			// optimistic UI update
			const release = await todos.override((todos) => {
				return [...todos, { id: '0', text, done: false }]
			})

			try {
				// submit form
				await submit()
			} finally {
				// remove override
				release()
				// clear form
				form.reset()
			}
		})}
	>
		<input type="text" name="text" placeholder="Add todo" autocomplete="off" />
		<button type="submit">Add</button>
		{#if addTodo.error}
			<p class="error">{addTodo.error.message}</p>
		{/if}
	</form>

	<ul>
		<!-- async svelte ❤️ -->
		{#each await todos as todo}
			<!-- useful when you have multiple forms that use the same remote form action for reuse -->
			{@const remove = deleteTodo.for(todo.id)}

			<li>
				<label>
					<input
						type="checkbox"
						checked={todo.done}
						onchange={async () => {
							// this should be a form but we want to showcase using commands
							const release = await todos.override((todos) => {
								return todos.map((t) => (t.id === todo.id ? { ...t, done: !t.done } : t))
							})

							try {
								await toggleTodo(todo.id)
								// here `toggleTodo` doesn't do a single flight mutation, so we refresh on the client
								await todos.refresh()
							} finally {
								// remove override
								release()
							}
						}}
					/>
					{#if !todo.editing}
						<span class={{ done: todo.done }}>{todo.text}</span>
					{:else}
						<input
							type="text"
							value={todo.text}
							onchange={async (e) => {
								const target = e.target as HTMLInputElement | null
								const text = target ? target.value.trim() : ''

								// optimistic UI update
								const release = await todos.override((todos) => {
									return todos.map((t) => (t.id === todo.id ? { ...t, text } : t))
								})

								try {
									await updateTodo(todo.id, text)
									// here `updateTodo` doesn't do a single flight mutation, so we refresh on the client
									await todos.refresh()
								} finally {
									// remove override
									release()
								}
							}}
						/>
					{/if}
				</label>

				<!-- using enhance to customize how the form is progressively enhanced -->
				<form
					{...remove.enhance(async ({ submit }) => {
						// optimistic UI update
						const release = await todos.override((todos) => {
							return todos.filter((t) => t.id !== todo.id)
						})

						try {
							// submit form
							await submit()
						} catch {
							// we catch the error to show the error inline instead of the nearest error page
						} finally {
							// remove override
							release()
						}
					})}
				>
					<!-- this seems to have bugs 🐛  -->
					{#if remove.error}
						<span class="error">{remove.error.message}</span>
					{/if}
					<!-- toggleEditing btn -->
					<button
						type="button"
						onclick={async () => {
							const release = await todos.override((todos) => {
								return todos.map((t) => (t.id === todo.id ? { ...t, editing: !t.editing } : t))
							})

							try {
								await toggleEditing(todo.id)
								// here `toggleEditing` doesn't do a single flight mutation, so we refresh on the client
								await todos.refresh()
							} finally {
								// remove override
								release()
							}
						}}
					>
						Edit
					</button>
					<button name="id" value={todo.id}>Delete</button>
				</form>
			</li>
		{/each}
	</ul>
</main>

<style>
	main {
		/* min-width: 600px; */
		margin: 0 auto;
		padding: 1rem;
	}

	ul {
		list-style: none;
		padding: 0;
	}
	li {
		display: grid;
		align-items: center;
		grid-template-columns: 1fr auto;
		gap: 1rem;
		padding: 1rem 0;
	}

	.done {
		text-decoration: line-through;
	}

	.error {
		color: red;
	}
</style>
