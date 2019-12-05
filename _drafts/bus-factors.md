Fix up the script to compute the bus factor as per

https://peerj.com/preprints/1233/

Ideas:

- short-term bus factor: committers per top-level subsystem

- long-term bus factor: reviewers per top-level subsystem

- normalize bus factor per # commits

- drill into drm

- try to measure the pipeline from reviewer to full maintainer/committer? E.g.
  reviewers per committers - might need a timeline approach.
