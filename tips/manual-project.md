# Create "Manual" type project

The Git repository is the most preferred place to store your playbooks, and [the guides and resources to deploy Private Git Repository on K3s](../git) are also provided in this repository.

However, if you don't need such a rich version control system, or need to make frequent iterations of trial and error to develop new playbooks, creating a project with **Manual** type is very helpful.

## Concepts

The project directories have to exist under `/var/lib/awx/projects` of AWX, and if you have deployed AWX following the steps described in [the main guide on this repository](../README.md), `/data/projects` on your K3s host is mounted as `/var/lib/awx/projects` in AWX.

So, to add project directories, simply, just placing it under `/data/projects` on your K3s host.

## Procedure

Create new directory under `/data/projects` on your K3s host, and place your playbooks under the directory you created.

```bash
$ tree /data/projects/
/data/projects/
`-- my-first-manual-project     👈👈👈
    `-- my-playbook.yaml     👈👈👈
```

Go to `Resources` > `Projects` > `Add` in AWX Web UI, fill `Name` field and select `Manual` as `Source Control Type`.

Now you can select your project directory (`my-first-manual-project` in this example) as `Playbook Directory`.

After `Save` the project, your playbooks can be selected in the Job Templates.

## Troubleshooting

If you got following warning while selecting `Playbook Directory`, try super-reloading your browser (`Shift + F5` or `Ctrl (Cmd) + Shift + R`) to refresh the page without using the cache stored in the browser.

> ⚠️WARNING:
> There are no available playbook directories in /var/lib/awx/projects
