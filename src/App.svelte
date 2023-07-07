<script lang="ts">
  import * as lua from 'lua-in-js/dist/lua-in-js.cjs.js'
  import NDK, { NDKNip07Signer, NDKEvent } from "@nostr-dev-kit/ndk"

  const defaultScript = `
  return {
    -- Filter out short events with lots of tags
    filterEvent = function (event)
      return #event.content < 300 and #event.tags > 5
    end
  }`.replace(/\n  /g, '\n').trim()

  const defaultTest = `
  const event = new NDKEvent(ndk, {
    kind: 1,
    content: "Hello world! This is a testing event for #nostrscript",
    tags: [["p", user.hexpubkey()], ["t", "nostrscript"]],
  })

  await event.sign()

  return myScript.filterEvent(event.rawEvent())
  `.replace(/\n  /g, '\n').trim()

  const first = xs => xs[0]

  const isNil = x => x === null || typeof x === 'undefined'

  // https://stackoverflow.com/a/12554727
  function sandbox(code, locals) {
    const params = []
    const args = []

    // create our own local versions of window and document with limited functionality
    locals = Object.assign({window: {}, document: {}}, locals)

    for (const [k, v] of Object.entries(locals)) {
      params.push(k)
      args.push(v)
    }

    const AsyncFunction = async function () {}.constructor
    const func = new AsyncFunction(...params.concat(code))

    return func.apply(Object.create(null), args)
  }

  const formatTimestamp = ts => {
    const formatter = new Intl.DateTimeFormat("en-US", {
      dateStyle: "medium",
      timeStyle: "short",
    })

    return formatter.format(new Date(ts * 1000))
  }

  const mapv = (f, o) => {
    const r = {}

    for (const [k, v] of Object.entries(o)) {
      r[k] = f(v)
    }

    return r
  }

  const wrap = x => {
    if (Array.isArray(x)) {
      return new lua.Table(x.map(wrap))
    }

    if (x && typeof x === 'object') {
      return new lua.Table(mapv(wrap, x))
    }

    return x
  }

  const unwrap = x => {
    if (typeof x === 'function') {
      return (...args) => unwrap(first(x(...args.map(wrap))))
    }

    if (typeof x !== 'object' || x === null) {
      return x
    }

    if (x.numValues?.[1]) {
      return x.numValues.slice(1).map(unwrap)
    }

    if (x.strValues && Object.entries(x.strValues).length > 0) {
      return mapv(unwrap, x.strValues)
    }

    return {}
  }

  const display = x => {
    if (typeof x === 'function') {
      return `<function>`
    }

    if (Array.isArray(x)) {
      x = x.map(display)
    }

    if (x && typeof x === 'object') {
      x = mapv(display, x)
    }

    return JSON.stringify(x, null, 2)
  }

  const cx = {
    error: "rounded bg-red-500 text-white",
    success: "rounded bg-green-500 text-white",
    border: "border-solid border border-blue-100",
    button: "rounded-full bg-blue-500 text-white",
    button2: "border-solid border rounded-full",
    button3: "border-solid border border-red-500 rounded-full",
    padding: "py-2 px-3",
  }

  let ndk
  let user
  let script
  let scripts = []
  let mode = 'editor'

  const showEditor = () => {
    mode = 'editor'
  }

  const showConsole = () => {
    mode = 'console'
    testScript()
  }

  const login = async () => {
    ndk = new NDK({
      signer: new NDKNip07Signer(),
      explicitRelayUrls: [
        "wss://nos.lol",
        "wss://nostr-pub.wellorder.net",
        "wss://relay.nostr.band",
      ],
    })

    user = first(await Promise.all([
      ndk.signer.user(),
      ndk.connect(),
    ]))

    loadScripts()
  }

  const loadScripts = async () => {
    const allScripts = Array.from(await ndk.fetchEvents({
      kinds: [37077],
      authors: [user.hexpubkey()],
      limit: 1000,
    }))

    const deletions = Array.from(await ndk.fetchEvents({
      kinds: [5],
      authors: [user.hexpubkey()],
      '#a': Array.from(allScripts).map(e => e.tagReference()[1]),
    }))

    const deletedAddresses = new Set(deletions.map(e => e.tagValue("a")))

    scripts = allScripts.filter(e => !deletedAddresses.has(e.tagReference()[1]))
  }

  const newScript = () => {
    script = {
      name: "My Script",
      content: defaultScript,
      description: "A simple nostr script",
      result: null,
      error: null,
      test: defaultTest,
    }

    validateScript()
  }

  const buildScript = async () => {
    const env = lua.createEnv()

    let result
    try {
      result = unwrap(env.parse(script.content).exec())
    } catch (e) {
      return [null, e.toString()]
    }

    if (!script.name) {
      return [null, "No script name provided"]
    }

    return [result, null]
  }

  const validateScript = async () => {
    const [result, error] = await buildScript()

    script.result = result
    script.error = error

    return Boolean(error)
  }

  const testScript = async () => {
    script.testResult = null
    script.testError = null
    try {
      script.testResult = await sandbox(script.test, {
        ndk,
        user,
        NDKEvent,
        myScript: script.result,
      })
    } catch (e) {
      script.testError = e.toString()
    }
  }

  const publishScript = async () => {
    if (!validateScript()) {
      return
    }

    const event = new NDKEvent(ndk, {
      kind: 37077,
      content: script.content,
      tags: [
        ["d", script.name],
        ["description", script.description],
      ],
    })

    await event.publish()
    await loadScripts()

    script = null
  }

  const editScript = async e => {
    script = {
      event: e,
      content: e.content,
      name: e.tagValue("d"),
      description: e.tagValue("description"),
    }

    validateScript()
  }

  const clearScript = async () => {
    script = null
  }

  const deleteScript = async () => {
    if (confirm("Are you sure you want to delete this script?")) {
      await script.event.delete()
      await loadScripts()

      script = null
    }
  }
</script>

<div class="p-8 m-auto flex flex-col gap-4 sm:w-3/4">
  <h1 class="text-2xl">OstrichScript Editor</h1>
  <div class={`${cx.border} rounded-2xl p-8 flex-grow flex flex-col gap-2`}>
    {#if user}
      {#if script}
        <div class="flex flex-col gap-2">
          <div class="grid grid-cols-4">
            <label for="title" class={cx.padding}>Title</label>
            <input
              name="title"
              class={`${cx.border} ${cx.padding} rounded col-span-3`}
              bind:value={script.name} />
          </div>
          <div class="grid grid-cols-4">
            <label for="description" class={cx.padding}>Description</label>
            <textarea
              rows="2"
              name="description"
              class={`${cx.border} ${cx.padding} rounded col-span-3`}
              bind:value={script.description}
              on:input={validateScript} />
          </div>
          <div class="flex gap-2">
            <button class:underline={mode === 'editor'} class={`${cx.padding} cursor-pointer`} on:click={showEditor}>Editor</button>
            <button class:underline={mode === 'console'} class={`${cx.padding} cursor-pointer`} on:click={showConsole}>Console</button>
          </div>
          {#if mode === 'editor'}
            <textarea
              rows="16"
              name="content"
              class={`${cx.border} ${cx.padding} rounded`}
              bind:value={script.content}
              on:input={validateScript} />
            {#if script.error || isNil(script.result)}
              <pre class={`${cx.error} ${cx.padding}`}>{script.error || "No value returned"}</pre>
            {/if}
            <div class="text-gray-700">
              OstrichScript uses the lua programming language to extend nostr clients. Read more about it
              <a href="https://github.com/coracle-social/ostrich-script" class="underline" target="_blank">here</a>.
            </div>
          {:else}
            <textarea
              rows="16"
              name="test"
              class={`${cx.border} ${cx.padding} rounded`}
              bind:value={script.test}
              on:input={testScript} />
            {#if script.testError}
              <pre class={`${cx.error} ${cx.padding}`}>{script.testError}</pre>
            {:else}
              <pre class={`${cx.success} ${cx.padding} overflow-auto`}>{display(script.testResult)}</pre>
            {/if}
          {/if}
          <div class="flex justify-end gap-2 mt-2">
            <button on:click={clearScript} class={`${cx.button2} ${cx.padding}`}>Go Back</button>
            {#if script.event}
              <button on:click={deleteScript} class={`${cx.button3} ${cx.padding}`}>Delete</button>
            {/if}
            <button on:click={publishScript} class={`${cx.button} ${cx.padding}`}>Publish</button>
          </div>
        </div>
      {:else}
        <div class="flex justify-between">
          <h2 class="text-xl font-bold">Your scripts</h2>
          <button on:click={newScript} class={`${cx.button} ${cx.padding}`}>Add Script</button>
        </div>
        <div class="flex flex-col gap-3 my-2">
          {#each scripts as e}
            <div class="flex items-center justify-between">
              <div class="flex gap-3">
                <span>{e.tagValue("d")}</span>
                <span class="text-gray-500">Created {formatTimestamp(e.created_at)}</span>
              </div>
              <button on:click={() => editScript(e)} class={`underline ${cx.padding}`}>Edit Script</button>
            </div>
          {:else}
            <p>No scripts found</p>
          {/each}
        </div>
      {/if}
    {:else}
      <div class="flex justify-center items-center h-full">
        <button on:click={login} class={`${cx.button} ${cx.padding}`}>Login with NIP 07</button>
      </div>
    {/if}
  </div>
</div>
