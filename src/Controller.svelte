<script lang="ts">
  import { EventType } from 'rrweb';
  import type { Replayer } from 'rrweb';
  import type { playerMetaData } from 'rrweb/typings/types';
  import type {
    PlayerMachineState,
    SpeedMachineState,
  } from 'rrweb/typings/replay/machine';
  import {
    onMount,
    onDestroy,
    createEventDispatcher,
    afterUpdate,
  } from 'svelte';
  import { formatTime } from './utils';
  import Switch from './components/Switch.svelte';

  const dispatch = createEventDispatcher();

  export let replayer: Replayer;
  export let showController: boolean;
  export let autoPlay: boolean;
  export let skipInactive: boolean;
  export let speedOption: number[];
  export let speed = speedOption.length ? speedOption[0] : 1;
  export let tags: Record<string, string> = {};

  let currentTime = 0;
  $: {
    dispatch('ui-update-current-time', { payload: currentTime });
  }
  let timer: number | null = null;
  let playerState: 'playing' | 'paused' | 'live';
  $: {
    dispatch('ui-update-player-state', { payload: playerState });
  }
  let speedState: 'normal' | 'skipping';
  let progress: HTMLElement;
  let step: HTMLElement;
  let finished: boolean;

  let meta: playerMetaData;
  $: meta = replayer.getMetaData();
  let percentage: string;
  $: {
    const percent = Math.min(1, currentTime / meta.totalTime);
    percentage = `${100 * percent}%`;
    dispatch('ui-update-progress', { payload: percent });
  }
  type CustomEvent = {
    name: string;
    background: string;
    position: string;
  };
  let customEvents: CustomEvent[];
  $: customEvents = (() => {
    const { context } = replayer.service.state;
    const totalEvents = context.events.length;
    const start = context.events[0].timestamp;
    const end = context.events[totalEvents - 1].timestamp;
    const customEvents: CustomEvent[] = [];

    // calculate tag position.
    const position = (startTime: number, endTime: number, tagTime: number) => {
      const sessionDuration = endTime - startTime;
      const eventDuration = endTime - tagTime;
      const eventPosition = 100 - (eventDuration / sessionDuration) * 100;

      return eventPosition.toFixed(2);
    };

    // loop through all the events and find out custom event.
    context.events.forEach((event) => {
      /**
       * we are only interested in custom event and calculate it's position
       * to place it in player's timeline.
       */
      if (event.type === EventType.Custom) {
        const customEvent = {
          name: event.data.tag,
          background: tags[event.data.tag] || 'rgb(73, 80, 246)',
          position: `${position(start, end, event.timestamp)}%`,
        };
        customEvents.push(customEvent);
      }
    });

    return customEvents;
  })();

  const loopTimer = () => {
    stopTimer();

    function update() {
      currentTime = replayer.getCurrentTime();

      if (currentTime < meta.totalTime) {
        timer = requestAnimationFrame(update);
      }
    }

    timer = requestAnimationFrame(update);
  };

  const stopTimer = () => {
    if (timer) {
      cancelAnimationFrame(timer);
      timer = null;
    }
  };

  export const toggle = () => {
    switch (playerState) {
      case 'playing':
        pause();
        break;
      case 'paused':
        play();
        break;
      default:
        break;
    }
  };

  export const play = () => {
    if (playerState !== 'paused') {
      return;
    }
    if (finished) {
      replayer.play();
      finished = false;
    } else {
      replayer.play(currentTime);
    }
  };

  export const pause = () => {
    if (playerState !== 'playing') {
      return;
    }
    replayer.pause();
  };

  export const goto = (timeOffset: number) => {
    currentTime = timeOffset;
    const isPlaying = playerState === 'playing';
    replayer.pause();
    replayer.play(timeOffset);
    if (!isPlaying) {
      replayer.pause();
    }
  };

  const handleProgressClick = (event: MouseEvent) => {
    if (speedState === 'skipping') {
      return;
    }
    const progressRect = progress.getBoundingClientRect();
    const x = event.clientX - progressRect.left;
    let percent = x / progressRect.width;
    if (percent < 0) {
      percent = 0;
    } else if (percent > 1) {
      percent = 1;
    }
    const timeOffset = meta.totalTime * percent;
    goto(timeOffset);
  };

  export const setSpeed = (newSpeed: number) => {
    let needFreeze = playerState === 'playing';
    speed = newSpeed;
    if (needFreeze) {
      replayer.pause();
    }
    replayer.setConfig({ speed });
    if (needFreeze) {
      replayer.play(currentTime);
    }
  };

  export const toggleSkipInactive = () => {
    skipInactive = !skipInactive;
  };

  onMount(() => {
    playerState = replayer.service.state.value as typeof playerState;
    speedState = replayer.speedService.state.value as typeof speedState;
    replayer.on(
      'state-change',
      (states: { player?: PlayerMachineState; speed?: SpeedMachineState }) => {
        const { player, speed } = states;
        if (player?.value && playerState !== player.value) {
          playerState = player.value as typeof playerState;
          switch (playerState) {
            case 'playing':
              loopTimer();
              break;
            case 'paused':
              stopTimer();
              break;
            default:
              break;
          }
        }
        if (speed?.value && speedState !== speed.value) {
          speedState = speed.value as typeof speedState;
        }
      },
    );
    replayer.on('finish', () => {
      finished = true;
    });

    if (autoPlay) {
      replayer.play();
    }
  });

  afterUpdate(() => {
    if (skipInactive !== replayer.config.skipInactive) {
      replayer.setConfig({ skipInactive });
    }
  });

  onDestroy(() => {
    replayer.pause();
    stopTimer();
  });
</script>

{#if showController}
  <div class="rr-controller">
    <div class="rr-timeline">
      <span class="rr-timeline__time">{formatTime(currentTime)}</span>
      <div
        class="rr-progress"
        class:disabled={speedState === 'skipping'}
        bind:this={progress}
        on:click={(event) => handleProgressClick(event)}>
        <div
          class="rr-progress__step"
          bind:this={step}
          style="width: {percentage}" />
        {#each customEvents as event}
          <div
            title={event.name}
            style="width: 10px;height: 5px;position: absolute;top:
            2px;transform: translate(-50%, -50%);background: {event.background};left:
            {event.position};" />
        {/each}

        <div class="rr-progress__handler" style="left: {percentage}" />
      </div>
      <span class="rr-timeline__time">{formatTime(meta.totalTime)}</span>
    </div>
  </div>

  <div class="align-player-controls">
    <div class="player-btn-groups btn-group" role="group">
      <button class="btn btn-md btn-primary" on:click={toggle}>
        {#if playerState === 'playing'}
          <i class="fa fa-pause"></i>
        {:else}
          <i class="fa fa-play"></i>
        {/if}
      </button>
      {#each speedOption as s}
        <button
          class="btn btn-md btn-primary"
          class:active={s === speed && speedState !== 'skipping'}
          on:click={() => setSpeed(s)}
          disabled={speedState === 'skipping'}>
          {s}x
        </button>
      {/each}
      <button class="btn btn-md btn-primary" title="Make Notes" on:click={() => dispatch('makenotes')}>
        <i class="fa fa-pen"></i>
      </button>
      <button class="btn btn-md btn-primary btn-fullscreen" on:click={() => dispatch('fullscreen')}>
        <i class="fa fa-expand"></i>
      </button>
      <Switch
        id="skip"
        bind:checked={skipInactive}
        disabled={speedState === 'skipping'}
        label="Skip Inactive" />
    </div>
  </div>
{/if}
