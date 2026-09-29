# Maintainer communication

Ask for help without outsourcing thinking: “I’m tracing upload cleanup through [view lines 44–60](../../mayan/apps/documents/api_views/document_file_api_views.py#L44-L60) and [task lines 90–124](../../mayan/apps/documents/tasks.py#L90-L124). I reproduced X, expected Y, tested A/B, and now suspect Z. Is orphan cleanup intentionally external, or would a failure-injection test be welcome?”

Bug report: version/environment → smallest reproduction → expected/actual → logs without secrets → suspected boundary → workaround → willingness to test. Feature proposal: user problem → current limitation with anchor → smallest useful behavior → alternatives → permission/data/ops impact. Review response: restate concern, show changed evidence, ask one precise question if still ambiguous. Respectful disagreement: separate requirement from implementation preference and propose a cheap experiment.
