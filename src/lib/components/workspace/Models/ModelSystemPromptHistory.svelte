<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { getContext, onMount } from 'svelte';
	import {
		getModelSystemPromptHistory,
		restoreModelSystemPromptVersion,
		deleteModelSystemPromptHistoryEntry
	} from '$lib/apis/models';
	import Spinner from '$lib/components/common/Spinner.svelte';
	import Badge from '$lib/components/common/Badge.svelte';
	import dayjs from 'dayjs';

	const i18n = getContext('i18n');

	export let modelId: string;
	export let versionId: string | null = null;
	export let onRestore: (system: string) => void;

	let history: any[] = [];
	let loading = false;
	let restoring = false;

	const loadHistory = async () => {
		if (!modelId) return;
		loading = true;
		try {
			history = (await getModelSystemPromptHistory(localStorage.token, modelId)) || [];
		} catch {
			history = [];
		}
		loading = false;
	};

	const handleRestore = async (entry: any) => {
		restoring = true;
		try {
			const updated = await restoreModelSystemPromptVersion(localStorage.token, modelId, entry.id);
			toast.success($i18n.t('System prompt version restored — already live, no need to save'));
			versionId = entry.system_prompt_version_id ?? entry.id;
			onRestore(entry.system_prompt);
		} catch (e) {
			toast.error(`${e}`);
		}
		restoring = false;
	};

	const handleDelete = async (entry: any) => {
		if (entry.id === versionId) {
			toast.error($i18n.t('Cannot delete the active version'));
			return;
		}
		try {
			await deleteModelSystemPromptHistoryEntry(localStorage.token, modelId, entry.id);
			toast.success($i18n.t('Version deleted'));
			await loadHistory();
		} catch (e) {
			toast.error(`${e}`);
		}
	};

	const renderDate = (timestamp: number) => {
		const d = timestamp * 1000;
		return dayjs(d).format('L LT');
	};

	onMount(loadHistory);
</script>

<div class="mt-2">
	<div class="flex items-center justify-between mb-1">
		<div class="text-xs font-medium text-gray-500">{$i18n.t('Version History')}</div>
		{#if restoring}
			<Spinner className="size-3" />
		{/if}
	</div>

	{#if loading}
		<div class="flex justify-center py-3">
			<Spinner className="size-4" />
		</div>
	{:else if history.length > 0}
		<div class="space-y-1 max-h-48 overflow-y-auto">
			{#each history as entry}
				<div
					class="flex items-center gap-2 px-3 py-1.5 rounded-lg {entry.id === versionId
						? 'bg-gray-100/50 dark:bg-gray-850/50'
						: 'hover:bg-gray-50 dark:hover:bg-gray-850'} transition"
				>
					<button
						class="flex-1 text-left text-xs truncate"
						on:click={() => handleRestore(entry)}
					>
						<div class="flex items-center gap-1">
							<span class="font-mono text-gray-500">{entry.id.slice(0, 7)}</span>
							{#if entry.id === versionId}
								<Badge type="success" content={$i18n.t('Live')} />
							{/if}
						</div>
						<div class="text-gray-400 truncate">{entry.commit_message || $i18n.t('Update')}</div>
						<div class="flex items-center gap-1 text-gray-400">
							{#if entry.user}
								<img
									src={`/api/v1/users/${entry.user.id}/profile/image`}
									alt={entry.user.name}
									class="size-3 rounded-full"
									on:error={(e) => (e.target.src = '/user.png')}
								/>
								<span class="truncate max-w-20">{entry.user.name}</span>
								<span>•</span>
							{/if}
							<span class="shrink-0">{renderDate(entry.created_at)}</span>
						</div>
					</button>
					{#if entry.id !== versionId}
						<button
							class="text-gray-400 hover:text-red-500 transition text-xs shrink-0"
							on:click={() => handleDelete(entry)}
							aria-label={$i18n.t('Delete version')}
						>
							<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" fill="currentColor" class="size-3.5">
								<path fill-rule="evenodd" d="M5 3.25V4H2.75a.75.75 0 0 0 0 1.5h.3l.815 8.15A1.5 1.5 0 0 0 5.357 15h5.286a1.5 1.5 0 0 0 1.492-1.35l.815-8.15h.3a.75.75 0 0 0 0-1.5H11v-.75A2.25 2.25 0 0 0 8.75 1h-1.5A2.25 2.25 0 0 0 5 3.25Zm2.25-.75a.75.75 0 0 0-.75.75V4h3v-.75a.75.75 0 0 0-.75-.75h-1.5ZM6.05 6a.75.75 0 0 1 .787.713l.275 5.5a.75.75 0 0 1-1.498.074l-.275-5.5A.75.75 0 0 1 6.05 6Zm3.9 0a.75.75 0 0 1 .712.787l-.275 5.5a.75.75 0 0 1-1.498-.074l.275-5.5a.75.75 0 0 1 .786-.713Z" clip-rule="evenodd"/>
							</svg>
						</button>
					{/if}
				</div>
			{/each}
		</div>
	{:else}
		<div class="text-xs text-gray-400 italic py-2">{$i18n.t('No version history yet')}</div>
	{/if}
</div>