<script lang="ts">
  export let disabled: boolean;
  export let checked: boolean;
  export let id: string;
  export let label: string;
</script>

<div class="cc-control custom-control custom-switch" class:disabled>
  <input type="checkbox" class="custom-control-input" {id} bind:checked {disabled}>
  <label class="custom-control-label" for={id}>
    {label}
  </label>
</div>
