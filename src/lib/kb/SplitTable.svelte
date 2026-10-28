<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type CauseGroup, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a merged ticket. causes.ts places a report by tenant; reports the desk
  // read as a one-day move (desk-notes.json) are not a month/day swap, so they get a row of their own.
  let { ticket, issues = {} }: { ticket: number; issues?: Record<string, string> } = $props();

  type Row = Omit<CauseGroup, 'cause'> & { problem: string };

  const rows = $derived.by(() => {
    const merged = uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket);
    const moved = new Set<string>(notes.oneDayMove);
    const summary = (problem: string, rs: Report[]): Row => {
      const [g] = groupByCause(rs, () => undefined);
      return { ...g, problem };
    };
    const out: Row[] = groupByCause(
      merged.filter((r) => !moved.has(r.id)),
      tenantOf,
    ).map((g) => ({ ...g, problem: g.cause }));
    const shifted = merged.filter((r) => moved.has(r.id));
    if (shifted.length) out.push(summary('one-day move', shifted));
    return { merged, rows: out.sort((a, b) => b.reports.length - a.reports.length) };
  });

  const tags = (g: { taggedAs: Record<string, number> }) =>
    Object.entries(g.taggedAs)
      .sort((a, b) => b[1] - a[1])
      .map(([tag, n]) => `${tag} ×${n}`)
      .join(' · ');
</script>

<p class="sum">
  <strong>{rows.merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.rows.length}</strong> problems
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tagged as</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows.rows as g (g.problem)}
      <tr>
        <td><code>{g.problem}</code></td>
        <td>{g.reports.length}</td>
        <td class="tags">{tags(g)}</td>
        <td>{g.tenants}</td>
        <td>{g.firstSeen} → {g.lastSeen}</td>
        <td>{#if issues[g.problem]}<a href={issues[g.problem]}>issue</a>{:else}none{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .tags { color: var(--muted); font-size: 0.85em; }
</style>
