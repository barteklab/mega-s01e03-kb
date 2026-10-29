<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem inside a merged ticket, each linked to its own ticket.
  // A report the desk read as a one-day move (desk-notes.json) is not a flipped date: it is its own problem.
  let { ticket, tickets }: { ticket: number; tickets: Record<string, string> } = $props();

  type Problem = 'parser' | 'hint' | 'one-day' | 'unplaced';

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const problemOf = (r: Report): Problem =>
    notes.oneDayMove.includes(r.id) ? 'one-day' : (causeOf(r, tenantOf(r)) ?? 'unplaced');

  const rows = $derived.by(() => {
    const by = new Map<Problem, Report[]>();
    for (const r of merged) by.set(problemOf(r), [...(by.get(problemOf(r)) ?? []), r]);
    return [...by.entries()]
      .map(([problem, rs]) => {
        const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
        return {
          problem,
          reports: rs.length,
          tenants: new Set(rs.map((r) => r.tenant)).size,
          firstSeen: days[0],
          lastSeen: days[days.length - 1],
        };
      })
      .sort((a, b) => b.reports - a.reports);
  });
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports in #{ticket} · <strong>{rows.length}</strong> problems ·
  <strong>{rows.reduce((n, r) => n + r.reports, 0)}</strong> reports placed
</p>

<table>
  <thead><tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr></thead>
  <tbody>
    {#each rows as r (r.problem)}
      <tr>
        <td><code>{r.problem}</code></td>
        <td>{r.reports}</td>
        <td>{r.tenants}</td>
        <td>{r.firstSeen} → {r.lastSeen}</td>
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
  .none { color: var(--warn); font-weight: 600; }
</style>
