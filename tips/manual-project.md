# Create "Manual" type project

The Git repository is the most preferred place to store your playbooks, and [the guides and resources to deploy Private Git Repository on K3s](../git) are also provided in this repository.

However, if you don't need such a rich version control system, or need to make frequent iterations of trial and error to develop new playbooks, creating a project with **Manual** type is very helpful.

## Concepts

The project directories have to exist under `/var/lib/awx/projects` of AWX, and if you have deployed AWX following the steps described in [the main guide on this repository](../README.md), `/data/projects` on your K3s host is mounted as `/var/lib/awx/projects` in AWX.

So, to add project directories, simply, just placing it under `/data/projects` on your K3s host.

## Procedure

Create new directory under `/data/projects` on your K3s host, and place your playbooks under the directory you created. Note that this directory and files under the directory must be readable by the user with UID `1000`.

```bash
$ tree /data/projects/
/data/projects/
`-- my-first-manual-project   👈👈👈
    `-- my-playbook.yaml      👈👈👈
```

Go to `Resources` > `Projects` > `Add` in AWX Web UI, fill `Name` field and select `Manual` as `Source Control Type`.

Now you can select your project directory (`my-first-manual-project` in this example) as `Playbook Directory`.

After `Save` the project, your playbooks can be selected in the Job Templates.

## Troubleshooting

If you got following warning while selecting `Playbook Directory`, try super-reloading your browser (`Shift + F5` or `Ctrl (Cmd) + Shift + R`) to refresh the page without using the cache stored in the browser.

> ⚠️WARNING:
> There are no available playbook directories in /var/lib/awx/projects

If super-reloading does not help you, ensure your playbooks are visible under project directory on `/var/lib/awx/projects` in awx-web pod.

- Ensure your project directory is visible
  - `kubectl -n awx exec -it deployment/awx-web -c awx-web -- ls -l /var/lib/awx/projects`
- Ensure your project directory contains at least one playbook
  - `kubectl -n awx exec -it deployment/awx-web -c awx-web -- ls -l /var/lib/awx/projects/<YOUR_PROJECT_DIRECTORY>`

> [!IMPORTANT]
> Any empty project directories and the directories that don't contain any valid playbooks will not be listed in UI. Also all playbooks have to be placed under project directory (means sub directory) on `/var/lib/awx/projects`, not directly under `/var/lib/awx/projects`.
