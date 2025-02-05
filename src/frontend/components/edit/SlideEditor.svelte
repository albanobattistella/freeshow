<script lang="ts">
    import type { MediaFit } from "../../../types/Main"

    import { activeEdit, activeShow, dictionary, media, outputs, showsCache } from "../../stores"
    import MediaLoader from "../drawer/media/MediaLoader.svelte"
    import { history } from "../helpers/history"
    import Icon from "../helpers/Icon.svelte"
    import { getActiveOutputs, getResolution } from "../helpers/output"
    import { getMediaFilter } from "../helpers/showActions"
    import { _show } from "../helpers/shows"
    import { getStyles } from "../helpers/style"
    import T from "../helpers/T.svelte"
    import Button from "../inputs/Button.svelte"
    import { getStyleResolution } from "../slide/getStyleResolution"
    import Zoomed from "../slide/Zoomed.svelte"
    import Center from "../system/Center.svelte"
    import DropArea from "../system/DropArea.svelte"
    import Snaplines from "../system/Snaplines.svelte"
    import Editbox from "./Editbox.svelte"

    $: currentShow = $activeShow?.id
    $: if (currentShow && $showsCache[currentShow] && $activeEdit.slide === null && _show("active").slides().get().length) activeEdit.set({ slide: 0, items: [] })
    $: ref = currentShow && $showsCache[currentShow] ? _show("active").layouts("active").ref()[0] : null
    $: Slide = $activeEdit.slide !== null && ref?.[$activeEdit.slide!] ? _show("active").slides([ref[$activeEdit.slide!]?.id]).get()[0] : null

    let lines: [string, number][] = []
    let mouse: any = null
    let newStyles: any = {}
    $: active = $activeEdit.items

    let width: number = 0
    let height: number = 0
    $: resolution = getResolution(Slide?.settings?.resolution, $outputs)
    // TODO: zoom more in...

    let ratio: number = 1

    // get backgruond
    $: bgId = ref?.[$activeEdit.slide!]?.data.background

    // get ghost background
    $: if (!bgId) {
        ref?.forEach((a, i) => {
            if (i <= $activeEdit.slide! && !a.data.disabled) {
                if (a.data.actions?.clearBackground) bgId = null
                else if (a.data.background) bgId = a.data.background
            }
        })
    }

    $: background = bgId && currentShow ? $showsCache[currentShow].media[bgId] : null
    // $: slideOverlays = ref?.[$activeEdit.slide!]?.data.overlays || []

    let filter: string = ""
    let flipped: boolean = false
    let fit: MediaFit = "contain"

    $: if (background?.path) {
        // TODO: use show filter if existing
        filter = getMediaFilter(background.path)
        flipped = $media[background.path]?.flipped || false
        fit = $media[background.path]?.fit || "contain"
    }

    $: {
        if (active.length) updateStyles()
        else newStyles = {}
    }

    function updateStyles() {
        if (!Object.keys(newStyles).length) return

        let items = $showsCache[$activeShow?.id!].slides[ref[$activeEdit.slide!]?.id].items
        let values: any[] = []
        active.forEach((id) => {
            let item = items[id]
            if (item) {
                let styles: any = getStyles(item.style)
                let textStyles: string = ""

                Object.entries(newStyles).forEach(([key, value]: any) => (styles[key] = value))
                Object.entries(styles).forEach((obj) => (textStyles += obj[0] + ":" + obj[1] + ";"))

                values.push(textStyles)
            }
        })

        history({
            id: "setStyle",
            newData: { style: { key: "style", values } },
            location: { page: "edit", show: $activeShow!, slide: ref[$activeEdit.slide!].id, items: active },
        })
    }

    // $: if (Object.keys(newStyles).length && $showsCache[$activeShow?.id!] && active.length) {
    //   // let items = $showsCache[$activeShow?.id!].slides[ref[$activeEdit.slide!].id].items
    //   let items = _show("active").slides([ref[$activeEdit.slide!].id]).items().get()[0]
    //   if (items) autoSize(active, items)
    // }

    let altKeyPressed: boolean = false
    function keydown(e: any) {
        if (e.altKey) {
            e.preventDefault()
            altKeyPressed = true
        }
    }
    function keyup() {
        altKeyPressed = false
    }

    let zoom = 1

    function wheel(e: any) {
        if (!e.ctrlKey && !e.metaKey) return
        if (!e.target.closest(".editArea")) return
        zoom = Math.max(0.5, Math.min(2, zoom + (e.deltaY < 0 ? -0.1 : 0.1)))
    }

    // CHORDS
    let chordsMode: boolean = false
    function toggleChords() {
        chordsMode = !chordsMode
    }
</script>

<svelte:window on:keydown={keydown} on:keyup={keyup} on:mousedown={keyup} on:wheel={wheel} />

<div class="editArea">
    <!-- zoom: {1 / zoom}; -->
    <!-- width: {100 / zoom}%;height: {100 / zoom}%; -->
    <div class="parent" bind:offsetWidth={width} bind:offsetHeight={height}>
        {#if Slide}
            <DropArea id="edit">
                <Zoomed
                    background={Slide?.settings?.color || $outputs[getActiveOutputs()[0]]?.show?.background || "black"}
                    {resolution}
                    style={getStyleResolution(resolution, width, height, "fit", { zoom })}
                    bind:ratio
                    hideOverflow={false}
                    center={zoom >= 1}
                >
                    <!-- <div class="chordsButton" style="zoom: {1 / ratio};">
                        <Button on:click={toggleChords}>
                            <Icon id="chords" white={!chordsMode} />
                        </Button>
                    </div> -->
                    <!-- background -->
                    {#if !altKeyPressed && background}
                        {#key background.path}
                            <div class="background" style="zoom: {1 / ratio};opacity: 0.5;">
                                <MediaLoader path={background.path || background.id || ""} type={background.type !== "player" ? background.type : null} {filter} {flipped} {fit} />
                            </div>
                        {/key}
                    {/if}
                    <!-- edit -->
                    <Snaplines bind:lines bind:newStyles bind:mouse {ratio} {active} />
                    {#each Slide.items as item, index}
                        <Editbox {item} {chordsMode} ref={{ showId: currentShow, id: Slide.id }} {index} {ratio} bind:mouse />
                    {/each}
                    <!-- overlays -->
                    <!-- {#if !altKeyPressed && slideOverlays?.length}
            <div style="opacity: 0.5;pointer-events: none;">
              {#each slideOverlays as id}
                {#if $overlays[id]}
                  {#each $overlays[id].items as item}
                    <Textbox {item} ref={{ type: "overlay", id }} />
                  {/each}
                {/if}
              {/each}
            </div>
          {/if} -->
                </Zoomed>
            </DropArea>
        {:else}
            <Center size={2} faded>
                <T id="empty.slide" />
            </Center>
        {/if}
    </div>
    <div class="actions">
        <Button on:click={toggleChords} title={$dictionary.edit?.chords}>
            <Icon id="chords" white={!chordsMode} />
        </Button>
        <div class="seperator" />
        <Button disabled={zoom >= 2} on:click={() => (zoom += 0.1)} title={$dictionary.actions?.zoomOut}>
            <Icon size={1.3} id="remove" white />
        </Button>
        <Button disabled={zoom <= 0.5} on:click={() => (zoom -= 0.1)} title={$dictionary.actions?.zoomIn}>
            <Icon size={1.3} id="add" white />
        </Button>
        <p class="text" on:click={() => (zoom = 1)}>{(100 / zoom).toFixed()}%</p>
    </div>
</div>

<style>
    .editArea {
        width: 100%;
        height: 100%;
        display: flex;
        flex-direction: column;
    }

    .parent {
        width: 100%;
        height: 100%;
        display: flex;
        /* justify-content: center;
        align-items: center; */
        /* padding: 10px; */
        overflow: auto;
    }

    /* .chordsButton {
        position: absolute;
        top: 0;
        right: 0;
        background-color: var(--focus);
    }
    .chordsButton :global(button) {
        padding: 10px !important;
        z-index: 3;
    } */

    .actions {
        display: flex;
        align-items: center;
        justify-content: right;
        width: 100%;
        background-color: var(--primary);
        border-top: 3px solid var(--primary-lighter);
    }

    .seperator {
        width: 3px;
        height: 100%;
        background-color: var(--primary-lighter);
        /* margin: 0 10px; */
    }

    .text {
        opacity: 0.8;
        text-align: right;
        width: 50px;
        margin-right: 10px;
    }
</style>
