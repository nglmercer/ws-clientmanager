<!-- src/routes/+page.svelte -->
<script lang="ts">
	import { onMount } from 'svelte';
	import { WsConnectionManager } from '../lib/WsConnectionManager';
	import type { ConnectionState, Message,ConnectionOptions } from '../lib/types';
	import ConnectionCard from './ConnectionCard.svelte';
	import ConnectionDetail from './ConnectionDetail.svelte';
	import { emitter } from '../lib/Emitter';

	// --- Estado Reactivo ---
	let manager: WsConnectionManager;
	let connections: ConnectionState[] = [];
	let messages = new Map<string, Message[]>();
	let selectedConnectionId: string | null = null;
	
	// Derivados del estado
	$: selectedConnection = connections.find(c => c.id === selectedConnectionId) || null;
	$: selectedConnectionMessages = (selectedConnectionId && messages.get(selectedConnectionId)) || [];

	const eventName = 'ws:create';
	onMount(() => {
		manager = new WsConnectionManager();

		// --- Suscribirse a eventos del Manager ---
		manager.on('connectionCreated', (state: ConnectionState) => {
			connections = [...connections, state];
			messages.set(state.id, []);
			// Seleccionar la conexión recién creada
			selectedConnectionId = state.id;
		});

		manager.on('connectionRemoved', (id: string) => {
			connections = connections.filter((c) => c.id !== id);
			messages.delete(id);
			if (selectedConnectionId === id) {
				selectedConnectionId = null;
			}
		});

		manager.on('connectionStatusChange', (state: ConnectionState) => {
			connections = connections.map((c) => (c.id === state.id ? state : c));
		});

		manager.on('message', (payload: { connectionId: string; message: Message }) => {
			const { connectionId, message } = payload;
			const currentMessages = messages.get(connectionId) || [];
			messages.set(connectionId, [...currentMessages, message]);
			messages = messages; // Forzar reactividad
		});
		emitter.on(eventName,(data)=>{
			handleCreate(data)
		})
		return () => {
			// Limpieza al desmontar el componente
			manager.disconnectAll();
			emitter.off(eventName,()=>{})
		}
	});
	console.log("emitter listener",)

	// --- Handlers de Eventos de Componentes Hijos ---
	function handleCreate(options:ConnectionOptions) {
		try {
			console.log("handleCreate",options)
			manager.createConnection(options);
		} catch (e) {
			console.log("err",e);
		}
	}

function handleConnect({ detail: id }) {
		manager.connect(id);
	}

	function handleDisconnect({ detail: id }) {
		manager.disconnect(id);
	}

	function handleRemove({ detail: id }) {
		manager.removeConnection(id);
	}

	function handleSelect({ detail: id }) {
		selectedConnectionId = id;
	}
	
	function handleSend({ detail: data }) {
		if (selectedConnectionId) {
			manager.send(selectedConnectionId, data);
		}
	}

</script>

<main>
	<h1>Svelte WebSocket Manager</h1>
	
	<div class="slot">
		<slot></slot>
	</div>

	<div class="layout">
		<aside class="connection-list">
			<h2>Conexiones</h2>
			{#if connections.length === 0}
				<p class="no-connections">No hay conexiones. ¡Crea una!</p>
			{:else}
				{#each connections as conn (conn.id)}
					<ConnectionCard
						connection={conn}
						isSelected={conn.id === selectedConnectionId}
						on:connect={handleConnect}
						on:disconnect={handleDisconnect}
						on:remove={handleRemove}
						on:select={handleSelect}
					/>
				{/each}
			{/if}
		</aside>

		<section class="detail-view">
			{#if selectedConnection}
				<ConnectionDetail 
					connection={selectedConnection}
					messages={selectedConnectionMessages}
					on:send={handleSend}
				/>
			{:else}
				<div class="placeholder">
					<p>Selecciona una conexión para ver sus detalles y mensajes.</p>
				</div>
			{/if}
		</section>
	</div>
</main>

<style>
	:global(body) {
		background-color: #1e1e1e;
		color: #f1f1f1;
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
	}
	main {
		max-width: 1400px;
		margin: 0 auto;
		padding: 1rem;
	}
	.layout {
		display: grid;
		grid-template-columns: 350px 1fr;
		gap: 1.5rem;
		height: 70vh;
	}
	.connection-list {
		display: flex;
		flex-direction: column;
		gap: 0.75rem;
		overflow-y: auto;
		padding-right: 10px;
	}
	.no-connections {
		color: #888;
		text-align: center;
		margin-top: 1rem;
	}
	.detail-view {
		height: 100%;
	}
	.placeholder {
		display: flex;
		align-items: center;
		justify-content: center;
		height: 100%;
		border: 2px dashed #444;
		border-radius: 8px;
		color: #888;
	}
</style>