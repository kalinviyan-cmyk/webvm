<script>
import WebVM from '$lib/WebVM.svelte';
import * as configObj from '/config_public_alpine';
import { tryPlausible } from '$lib/plausible.js';

function handleProcessCreated(processCount)
{
	if(processCount == 1)
	{
		tryPlausible("Alpine init");
	}
}
</script>

<WebVM configObj={configObj} processCallback={handleProcessCreated} cacheId="blocks_alpine">
	<p>Alpine Linux graphical desktop running in your browser, powered by CheerpX.</p>
</WebVM>
