# Reading a saved project (3MF), fallback

Names, part types, filaments, flushing, plate settings, height ranges and layer codes all have
commands (see SKILL.md's router). The file is only for what no reply carries.

A `.3mf` is a zip: `project_save {"path":"/abs/w/p.3mf"}`, then read the entry with a short script.
To change one, re-zip every member (not only the edited one; on macOS `sed -i ''` is BSD sed) and
`project_open {"path":"/abs/w/p2.3mf","discard_changes":true}`, auto-answering
`{"match":"unsaved changes","button":"Discard"}`. Tell the user you went this way.

- Brim ear positions: `Metadata/brim_ear_points.txt` (`object_list` gives the count).
- Variable layer height points: `Metadata/layer_heights_profile.txt` (`object_list` gives the count
  and extremes).
- Painted facets per triangle (supports, seam, colour): the `paint_*` attributes in the object's
  `3D/Objects/*.model`.
