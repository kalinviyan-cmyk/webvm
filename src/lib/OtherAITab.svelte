<script>
	import { onMount } from 'svelte';
	import { aiActivity } from './activities.js';
	import SmallButton from './SmallButton.svelte';
	export let handleTool;

	let provider = 'gemini';
	let apiKey = '';
	let keyReady = false;
	let prompt = '';
	let busy = false;
	let messages = [];
	let error = '';

	const providerInfo = {
		gemini: { name: 'Google Gemini', placeholder: 'Paste your Gemini API key', model: 'gemini-2.5-flash' },
		openai: { name: 'ChatGPT (OpenAI)', placeholder: 'Paste your OpenAI API key', model: 'gpt-4.1-mini' }
	};

	const openAITools = [
		{ type: 'function', function: { name: 'computer', description: 'Control the Linux desktop. Use only when a graphical desktop is available.', parameters: { type: 'object', properties: { action: { type: 'string', enum: ['screenshot','mouse_move','left_click','right_click','type','key','wait'] }, coordinate: { type: 'array', items: { type: 'integer' }, minItems: 2, maxItems: 2 }, text: { type: 'string' }, duration: { type: 'number' } }, required: ['action'] } } },
		{ type: 'function', function: { name: 'bash', description: 'Run a shell command in the Linux terminal.', parameters: { type: 'object', properties: { command: { type: 'string' } }, required: ['command'] } } }
	];
	const geminiTools = [{ functionDeclarations: [
		{ name: 'computer', description: 'Control the Linux desktop. Use only when a graphical desktop is available.', parameters: { type: 'OBJECT', properties: { action: { type: 'STRING', enum: ['screenshot','mouse_move','left_click','right_click','type','key','wait'] }, coordinate: { type: 'ARRAY', items: { type: 'INTEGER' } }, text: { type: 'STRING' }, duration: { type: 'NUMBER' } }, required: ['action'] } },
		{ name: 'bash', description: 'Run a shell command in the Linux terminal.', parameters: { type: 'OBJECT', properties: { command: { type: 'STRING' } }, required: ['command'] } }
	]}];

	onMount(() => {
		const savedProvider = localStorage.getItem('webvm-ai-provider');
		if (savedProvider === 'gemini' || savedProvider === 'openai') provider = savedProvider;
		loadKey();
	});

	function loadKey() {
		apiKey = localStorage.getItem('webvm-' + provider + '-api-key') || '';
		keyReady = apiKey.length > 0;
	}
	function changeProvider(value) {
		provider = value;
		localStorage.setItem('webvm-ai-provider', provider);
		messages = [];
		error = '';
		loadKey();
	}
	function saveKey() {
		if (!apiKey.trim()) return;
		apiKey = apiKey.trim();
		localStorage.setItem('webvm-' + provider + '-api-key', apiKey);
		keyReady = true;
		error = '';
	}
	function forgetKey() {
		localStorage.removeItem('webvm-' + provider + '-api-key');
		apiKey = '';
		keyReady = false;
		messages = [];
	}
	function push(role, text) {
		messages = [...messages, { role, text: String(text ?? '') }];
	}
	async function executeTool(name, args) {
		if (name === 'computer') return await handleTool({ action: args.action, coordinate: args.coordinate, text: args.text, duration: args.duration });
		if (name === 'bash') return await handleTool({ command: args.command });
		return 'Unsupported tool';
	}
	async function askOpenAI() {
		const apiMessages = [{ role: 'system', content: 'You are an assistant operating inside WebVM, a Linux virtual machine. Help the user and use the computer or bash tools when needed. Be careful with commands and explain important actions.' }];
		for (const m of messages) apiMessages.push({ role: m.role === 'assistant' ? 'assistant' : 'user', content: m.text });
		const response = await fetch('https://api.openai.com/v1/chat/completions', {
			method: 'POST', headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer ' + apiKey },
			body: JSON.stringify({ model: providerInfo.openai.model, messages: apiMessages, tools: openAITools, tool_choice: 'auto' })
		});
		const data = await response.json();
		if (!response.ok) throw new Error(data.error?.message || 'OpenAI request failed (' + response.status + ')');
		const message = data.choices?.[0]?.message;
		if (!message) throw new Error('OpenAI returned an empty response');
		if (message.content) push('assistant', message.content);
		if (message.tool_calls?.length) {
			apiMessages.push(message);
			for (const call of message.tool_calls) {
				let args = {};
				try { args = JSON.parse(call.function.arguments || '{}'); } catch {}
				push('tool', 'Running ' + call.function.name + '…');
				const result = await executeTool(call.function.name, args);
				apiMessages.push({ role: 'tool', tool_call_id: call.id, content: typeof result === 'string' ? result : JSON.stringify(result ?? 'Done') });
			}
			const follow = await fetch('https://api.openai.com/v1/chat/completions', {
				method: 'POST', headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer ' + apiKey },
				body: JSON.stringify({ model: providerInfo.openai.model, messages: apiMessages, tools: openAITools, tool_choice: 'auto' })
			});
			const followData = await follow.json();
			if (!follow.ok) throw new Error(followData.error?.message || 'OpenAI tool follow-up failed');
			const followMessage = followData.choices?.[0]?.message;
			if (followMessage?.content) push('assistant', followMessage.content);
		}
	}
	async function askGemini() {
		const contents = messages.filter(m => m.role !== 'tool').map(m => ({ role: m.role === 'assistant' ? 'model' : 'user', parts: [{ text: m.text }] }));
		const response = await fetch('https://generativelanguage.googleapis.com/v1beta/models/' + providerInfo.gemini.model + ':generateContent?key=' + encodeURIComponent(apiKey), {
			method: 'POST', headers: { 'Content-Type': 'application/json' },
			body: JSON.stringify({ systemInstruction: { parts: [{ text: 'You are an assistant operating inside WebVM, a Linux virtual machine. Help the user and use the computer or bash tools when needed. Be careful with commands and explain important actions.' }] }, contents, tools: geminiTools, toolConfig: { functionCallingConfig: { mode: 'AUTO' } } })
		});
		const data = await response.json();
		if (!response.ok) throw new Error(data.error?.message || 'Gemini request failed (' + response.status + ')');
		const parts = data.candidates?.[0]?.content?.parts || [];
		const textPart = parts.filter(p => p.text).map(p => p.text).join('\n');
		if (textPart) push('assistant', textPart);
		const calls = parts.filter(p => p.functionCall);
		if (calls.length) {
			const functionResponseParts = [];
			for (const part of calls) {
				const call = part.functionCall;
				push('tool', 'Running ' + call.name + '…');
				const result = await executeTool(call.name, call.args || {});
				functionResponseParts.push({ functionResponse: { name: call.name, response: { result: typeof result === 'string' ? result : JSON.stringify(result ?? 'Done') } } });
			}
			const follow = await fetch('https://generativelanguage.googleapis.com/v1beta/models/' + providerInfo.gemini.model + ':generateContent?key=' + encodeURIComponent(apiKey), {
				method: 'POST', headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({ systemInstruction: { parts: [{ text: 'You are an assistant operating inside WebVM, a Linux virtual machine.' }] }, contents: [...contents, data.candidates[0].content, { role: 'user', parts: functionResponseParts }], tools: geminiTools })
			});
			const followData = await follow.json();
			if (!follow.ok) throw new Error(followData.error?.message || 'Gemini tool follow-up failed');
			const followText = followData.candidates?.[0]?.content?.parts?.filter(p => p.text).map(p => p.text).join('\n');
			if (followText) push('assistant', followText);
		}
	}
	async function send() {
		const value = prompt.trim();
		if (!value || busy || !keyReady) return;
		prompt = '';
		push('user', value);
		busy = true;
		aiActivity.set(true);
		error = '';
		try {
			if (provider === 'openai') await askOpenAI();
			else await askGemini();
		} catch (e) {
			error = e.message || 'Request failed';
			push('error', error);
		} finally {
			busy = false;
			aiActivity.set(false);
		}
	}
	function handleKeydown(e) {
		if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); send(); }
	}
</script>

<h1 class="text-lg font-bold">AI Integration</h1>
<p>Choose an AI provider. API keys are stored only in this browser's local storage.</p>
<label class="text-sm">Provider</label>
<select class="bg-neutral-700 p-2 rounded-md" bind:value={provider} on:change={() => changeProvider(provider)}>
	<option value="gemini">Google Gemini</option>
	<option value="openai">ChatGPT (OpenAI API)</option>
</select>
{#if !keyReady}
	<p>{providerInfo[provider].name} requires an API key.</p>
	<textarea class="bg-neutral-700 p-2 rounded-md placeholder-gray-400 resize-none shrink-0" bind:value={apiKey} placeholder={providerInfo[provider].placeholder} rows="2"></textarea>
	<SmallButton buttonIcon="fa-solid fa-key" buttonText="Save API key" clickHandler={saveKey}/>
{:else}
	<div class="flex items-center gap-2">
		<span class="text-sm text-green-300">API key saved for {providerInfo[provider].name}</span>
		<SmallButton buttonIcon="fa-solid fa-trash" buttonText="Forget key" clickHandler={forgetKey}/>
	</div>
{/if}
<div class="flex grow flex-col overflow-y-auto gap-2 min-h-0">
	{#each messages as msg}
		<p class="p-2 rounded-md whitespace-pre-wrap {msg.role === 'user' ? 'bg-neutral-700' : msg.role === 'error' ? 'bg-red-900' : 'bg-neutral-800'}"><strong>{msg.role === 'user' ? 'You' : msg.role === 'tool' ? 'WebVM' : msg.role === 'error' ? 'Error' : providerInfo[provider].name}:</strong> {msg.text}</p>
	{/each}
</div>
{#if keyReady}
	<textarea class="bg-neutral-700 p-2 rounded-md placeholder-gray-400 resize-none shrink-0" bind:value={prompt} on:keydown={handleKeydown} placeholder={busy ? 'AI is working…' : 'Message… (Enter to send, Shift+Enter for a new line)'} rows="2" disabled={busy || handleTool === null}></textarea>
	<SmallButton buttonIcon="fa-solid fa-paper-plane" buttonText={busy ? 'Working…' : 'Send'} clickHandler={send}/>
{/if}
{#if error}<p class="text-red-300 text-sm">{error}</p>{/if}
<p class="text-xs text-gray-400">Models: Gemini 2.5 Flash and GPT-4.1 mini. API usage is billed by the provider separately from ChatGPT subscriptions.</p>
