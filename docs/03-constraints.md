# Standard 2: Constraints Define WHERE and HOW — Not WHAT

This is the thing that gets misunderstood about architectural constraints.

People hear "you can't import from a sibling module" and think: restriction. Limitation. Something in the way.

That's not what it is.

A constraint that says "business logic goes in `service.py`, database tables go in `orm.py`, public contracts go in `interfaces.py`" doesn't limit what you can build. It answers a question that would otherwise be asked again and again by every person — and every agent — that touches the codebase.

**Where does this code go?** The constraint answers it. Always the same answer. The question stops being asked.

For human engineers, this saves time and reduces friction. For AI agents, it's more fundamental: it collapses the search space. An agent that doesn't know where something belongs has to explore the entire codebase, make inferences, and guess. An agent that knows "service files contain business logic, and service files are always at `modules/<name>/service.py`" goes directly to the right place. Every time. For every module.
