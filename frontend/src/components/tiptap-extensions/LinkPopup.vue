<template>
    <div
        v-if="searchEndpoint"
        class="flex w-[30rem] max-w-[calc(100vw-1.5rem)] flex-col gap-2 rounded-lg border border-outline-gray-2 bg-surface-modal p-2.5 text-ink-gray-9 shadow-xl"
    >
        <label class="sr-only" for="wiki-link-search-input">
            Cikk keresése vagy URL beillesztése
        </label>
        <TextInput
            id="wiki-link-search-input"
            ref="inputRef"
            v-model="editUrl"
            type="text"
            class="w-full"
            placeholder="Cikk keresése vagy URL beillesztése"
            autocomplete="off"
            role="combobox"
            aria-autocomplete="list"
            :aria-controls="resultsId"
            :aria-expanded="Boolean(results.length)"
            :aria-activedescendant="activeDescendant"
            @keydown.down.prevent="moveActiveResult(1)"
            @keydown.up.prevent="moveActiveResult(-1)"
            @keydown.enter.prevent="submitSmartLink"
            @keydown.escape.prevent="cancelEdit"
        />

        <div
            v-if="status"
            class="px-1 text-sm text-ink-gray-5"
            role="status"
            aria-live="polite"
        >
            {{ status }}
        </div>

        <div
            v-if="results.length"
            :id="resultsId"
            class="max-h-72 overflow-y-auto rounded-md py-0.5"
            role="listbox"
        >
            <button
                v-for="(result, index) in results"
                :id="resultId(index)"
                :key="result.route"
                type="button"
                role="option"
                :aria-selected="index === activeIndex"
                :class="[
                    'flex w-full flex-col gap-0.5 rounded-md px-2.5 py-2 text-left outline-none',
                    index === activeIndex
                        ? 'bg-surface-gray-3'
                        : 'hover:bg-surface-gray-2',
                ]"
                @mouseenter="activeIndex = index"
                @mousedown.prevent
                @click="selectResult(result)"
            >
                <span class="text-base font-medium text-ink-gray-9">
                    {{ result.title || result.route }}
                </span>
                <span
                    v-if="resultMeta(result)"
                    class="text-sm text-ink-gray-5"
                >
                    {{ resultMeta(result) }}
                </span>
                <span
                    v-if="result.excerpt"
                    class="line-clamp-2 text-sm leading-5 text-ink-gray-7"
                >
                    {{ result.excerpt }}
                </span>
            </button>
        </div>

        <div
            class="flex items-center justify-end gap-2 border-t border-outline-gray-1 pt-2"
        >
            <Button
                v-if="href"
                variant="subtle"
                class="mr-auto text-ink-red-3"
                @click="removeLink"
            >
                Hivatkozás eltávolítása
            </Button>
            <Button variant="subtle" @click="cancelEdit">Mégse</Button>
            <Button variant="solid" @click="submitSmartLink">
                Hivatkozás mentése
            </Button>
        </div>
    </div>

    <div
        v-else
        class="flex w-72 items-center gap-2 rounded-lg border border-outline-gray-2 bg-surface-white p-2 shadow-xl"
    >
        <TextInput
            v-if="isEditing"
            ref="inputRef"
            v-model="editUrl"
            type="text"
            class="w-full"
            placeholder="https://example.com"
            @keydown.enter="saveLink"
            @keydown.escape="cancelEdit"
        />
        <a
            v-else
            class="flex-1 truncate pl-1 text-sm text-ink-gray-7 underline"
            :title="currentHref"
            :href="currentHref"
            target="_blank"
        >
            {{ currentHref }}
        </a>
        <div class="ml-auto flex shrink-0 items-center gap-1.5">
            <template v-if="isEditing">
                <Button
                    @click="saveLink"
                    title="Submit"
                    variant="subtle"
                >
                    <template #icon>
                        <LucideCheck class="size-4" />
                    </template>
                </Button>
                <Button
                    @click="cancelEdit"
                    title="Cancel"
                    variant="subtle"
                >
                    <template #icon>
                        <LucideX class="size-4" />
                    </template>
                </Button>
            </template>
            <template v-else>
                <Button
                    @click="copyLink"
                    title="Copy"
                    variant="subtle"
                >
                    <template #icon>
                        <LucideCopy class="size-4" />
                    </template>
                </Button>
                <Button
                    @click="startEditing"
                    title="Edit"
                    variant="subtle"
                >
                    <template #icon>
                        <LucidePencil class="size-4" />
                    </template>
                </Button>
                <Button
                    @click="removeLink"
                    title="Remove"
                    variant="subtle"
                >
                    <template #icon>
                        <LucideLink2Off class="size-4" />
                    </template>
                </Button>
            </template>
        </div>
    </div>
</template>

<script setup>
import { Button, TextInput, toast } from 'frappe-ui';
import {
	computed,
	nextTick,
	onBeforeUnmount,
	onMounted,
	ref,
	watch,
} from 'vue';
import LucideCheck from '~icons/lucide/check';
import LucideCopy from '~icons/lucide/copy';
import LucideLink2Off from '~icons/lucide/link-2-off';
import LucidePencil from '~icons/lucide/pencil';
import LucideX from '~icons/lucide/x';

const props = defineProps({
	href: {
		type: String,
		default: '',
	},
	isNew: {
		type: Boolean,
		default: false,
	},
	searchEndpoint: {
		type: String,
		default: '',
	},
});

const emit = defineEmits(['save', 'remove', 'cancel']);

const inputRef = ref(null);
const isEditing = ref(props.isNew || props.href === '');
const editUrl = ref(props.href || '');
const currentHref = ref(props.href || '');
const results = ref([]);
const activeIndex = ref(-1);
const status = ref('');
const resultsId = 'wiki-link-search-results';
const activeDescendant = computed(() =>
	activeIndex.value >= 0 ? resultId(activeIndex.value) : undefined,
);

let debounceTimer;
let requestController;

function resultId(index) {
	return ['wiki-link-search-result-', index].join('');
}

function resultMeta(result) {
	return [result.space, result.section].filter(Boolean).join(' · ');
}

function isValidUrl(url) {
	if (!url || /^javascript:/i.test(url)) return false;
	if (url.startsWith('/') || url.startsWith('#')) return true;
	if (/^(?:https?:|mailto:|tel:)/i.test(url)) {
		try {
			new URL(url, window.location.origin);
			return true;
		} catch {
			return false;
		}
	}
	return /^[\p{L}\p{N}](?:[\p{L}\p{N}-]*\.)+[\p{L}]{2,}(?:[/:?#].*)?$/u.test(
		url,
	);
}

function normalizeUrl(value) {
	const url = value.trim();
	if (!isValidUrl(url)) return '';
	if (
		url.startsWith('/') ||
		url.startsWith('#') ||
		/^(?:https?:|mailto:|tel:)/i.test(url)
	) {
		return url;
	}
	return ['https://', url].join('');
}

function clearSearch() {
	window.clearTimeout(debounceTimer);
	requestController?.abort();
	requestController = undefined;
	results.value = [];
	activeIndex.value = -1;
}

function scheduleSearch() {
	if (!props.searchEndpoint) return;
	clearSearch();
	const query = editUrl.value.trim();
	status.value = '';
	if (!query || isValidUrl(query)) return;
	if (query.length < 2) {
		status.value = 'Írj be legalább 2 karaktert!';
		return;
	}
	status.value = 'Keresés…';
	debounceTimer = window.setTimeout(() => searchArticles(query), 250);
}

async function searchArticles(query) {
	requestController?.abort();
	requestController = new AbortController();
	try {
		const endpoint = new URL(props.searchEndpoint, window.location.origin);
		endpoint.searchParams.set('q', query);
		endpoint.searchParams.set('limit', '8');
		const response = await fetch(endpoint, {
			credentials: 'same-origin',
			headers: { Accept: 'application/json' },
			signal: requestController.signal,
		});
		if (!response.ok) throw new Error('Knowledge Base search failed');
		const payload = await response.json();
		if (editUrl.value.trim() !== query) return;
		results.value = Array.isArray(payload.message)
			? payload.message.slice(0, 8)
			: [];
		activeIndex.value = results.value.length ? 0 : -1;
		status.value = results.value.length
			? [results.value.length, ' találat'].join('')
			: 'Nincs találat.';
	} catch (error) {
		if (error.name === 'AbortError') return;
		results.value = [];
		activeIndex.value = -1;
		status.value = 'A keresés most nem sikerült. Próbáld újra.';
	}
}

function moveActiveResult(step) {
	if (!results.value.length) return;
	activeIndex.value =
		(activeIndex.value + step + results.value.length) % results.value.length;
	nextTick(() => {
		document
			.getElementById(resultId(activeIndex.value))
			?.scrollIntoView({ block: 'nearest' });
	});
}

function selectResult(result) {
	if (!result?.route) return;
	editUrl.value = result.route;
	emit('save', result.route);
}

function submitSmartLink() {
	const activeResult = results.value[activeIndex.value];
	if (activeResult) {
		selectResult(activeResult);
		return;
	}
	saveLink();
}

function startEditing() {
	editUrl.value = currentHref.value;
	isEditing.value = true;
	focusInput();
}

function saveLink() {
	const url = normalizeUrl(editUrl.value);
	if (!url && editUrl.value.trim()) return;
	currentHref.value = url;
	isEditing.value = false;
	emit('save', url);
}

function cancelEdit() {
	if (props.searchEndpoint) {
		emit('cancel');
	} else if (props.href) {
		isEditing.value = false;
		editUrl.value = currentHref.value;
	} else {
		emit('save', '');
	}
}

function removeLink() {
	emit('remove');
}

async function copyLink() {
	if (currentHref.value) {
		try {
			await navigator.clipboard.writeText(currentHref.value);
			toast.success('Link copied');
		} catch {
			toast.error('Failed to copy');
		}
	}
}

async function focusInput() {
	await nextTick();
	if (inputRef.value?.el) {
		inputRef.value.el.focus();
		inputRef.value.el.select();
	}
}

watch(
	() => props.href,
	(newHref) => {
		currentHref.value = newHref || '';
		editUrl.value = newHref || '';
		isEditing.value = newHref === '';
	},
);
watch(editUrl, scheduleSearch);

onMounted(focusInput);
onBeforeUnmount(clearSearch);

defineExpose({
	startEditing,
});
</script>
