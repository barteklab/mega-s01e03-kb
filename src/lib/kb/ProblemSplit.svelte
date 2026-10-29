<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem inside a merged ticket, each linked to its own ticket. Counted the desk's way
  // (CauseTable): every report under the cause it is filed by. The one-day moves (desk-notes.json) are
  // their own problem, counted from inside the cause they are filed under.
  let { ticket, tickets }: { ticket: number; tickets: Record<string, string> } = $props();

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const groups = $derived(groupByCause(merged, tenantOf));

  const rows = $derived([
    ...groups.map((g) => ({
      problem: g.cause as string,
      reports: g.reports.length,
      tenants: g.tenants,
      seen: `${g.firstSeen} → ${g.lastSeen}`,
      within: '',
    })),
    ...groups.flatMap((g) => {
      const moved = g.reports.filter((r) => notes.oneDayMove.includes(r.id));
      if (!readAs(g, notes.oneDayMove)) return [];
      const days = moved.map((r) => r.receivedAt.slice(0, 10)).sort();
      return [{
        problem: 'one-day',
        reports: moved.length,
        tenants: new Set(moved.map((r) => r.tenant)).size,
        seen: `${days[0]} → ${days[days.length - 1]}`,
        within: g.cause,
      }];
    }),
  ]);
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports in #{ticket} · <strong>{rows.length}</strong> problems
</p>

<table>
  <thead><tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr></thead>
  <tbody>
    {#each rows as r (r.problem)}
      <tr>
        <td><code>{r.problem}</code>{#if r.within}<span class="within"> filed under <code>{r.within}</code></span>{/if}</td>
        <td>{r.reports}</td>
        <td>{r.tenants}</td>
        <td>{r.seen}</td>
        <td>
          {#if tickets[r.problem]}<a href={tickets[r.problem]}>#{tickets[r.problem].split('/').pop()}</a>
          {:else}<span class="none">no ticket</span>{/if}
        </td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .within { color: var(--muted); font-size: 0.85em; }
  .none { color: var(--warn); font-weight: 600; }
</style>
