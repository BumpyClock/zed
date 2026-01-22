# Learnings

## 2026-01-21: Blade Renderer Borrow Checker Issues

**Context**: Rebasing backdrop blur feature onto latest upstream caused borrow checker errors in the blade renderer.

**What was tried**:
- Original code tried to pass `&self.pipelines.backdrop_blur_downsample` (immutable borrow) while also calling `self.draw_backdrop_blur_pass(&mut self)` (mutable borrow)
- Original code also tried to call `self.backdrop_blur_passes_for_radius()` while a render pass (borrowing `self.command_encoder`) was still active

**Outcome**: Borrow checker errors at compile time

**Solution**:
1. Created `BackdropBlurDirection` enum instead of passing pipeline reference directly - this avoids the mutable/immutable borrow conflict since the method selects the pipeline internally after taking `&mut self`
2. Pre-computed pass counts for all blurs into a `Vec<usize>` before entering the render loop, allowing method calls on `self` while no borrows are active
3. Used a local `inner_pass` variable for each blur batch instead of reusing the main `pass` variable
4. Always recreate the main `pass` at the end of BackdropBlurs handling to ensure it's valid for subsequent batches

**Next time**: When working with Rust render encoders that borrow `self` fields mutably:
- Pre-compute any values that require `&self` method calls before creating the render pass
- Use enums or indices instead of references when a method needs both `&mut self` and a reference to a field
- Ensure render passes are always properly recreated after being dropped for complex control flow
