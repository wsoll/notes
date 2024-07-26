# Cheat sheet
```bash
git reset --soft HEAD~1
git remote -v
git branch -a
git log -n 5
git push --force-with-lease
```

# Commits
- https://www.conventionalcommits.org/en/v1.0.0/

# Merge strategies

<div style="border: 2px solid #ccc; padding: 10px; margin: 10px 0; border-radius: 5px; background-color: #f9f9f9;">
<p><strong>Merge Commit</strong></p>
<pre><code>A---B---C---D (main)
\       \
 E---F---G (feature)

When merged:
A---B---C---D---M (main)
 \           /
  E---F---G (feature)
</code></pre>
</div>

<div style="border: 2px solid #ccc; padding: 10px; margin: 10px 0; border-radius: 5px; background-color: #f9f9f9;">
<p><strong>Merge Commit with Semi-Linear History</strong></p>
<pre><code>Before rebase:
A---B---C---D (main)
 \       \
  E---F---G (feature)
After rebase:
A---B---C---D (main)
            \
             E'---F'---G' (feature)
After merge:
A---B---C---D---M (main)
                \
                 E'---F'---G' (feature)
</code></pre>
</div>

<div style="border: 2px solid #ccc; padding: 10px; margin: 10px 0; border-radius: 5px; background-color: #f9f9f9;">
<p><strong>Fast-Forward Merge</strong></p>
<pre><code>A---B---C (main)
      \
       D---E---F (feature)

After merge:
A---B---C---D---E---F (main)
</code></pre>
</div>
