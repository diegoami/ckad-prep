# Resources

What I used (or would use again) to prepare, grouped by purpose.

## Official

| Resource | Notes |
|---|---|
| [CKAD curriculum](https://github.com/cncf/curriculum) | The exam's scope and domain weights. Short; read it twice. |
| [CKA/CKAD exam tips](https://docs.linuxfoundation.org/tc-docs/certification/tips-cka-and-ckad) | The Linux Foundation's own advice: environment, allowed tools, and so on. |
| [Resources allowed during the exam](https://docs.linuxfoundation.org/tc-docs/certification/certification-resources-allowed) | Which docs you can open in the exam browser. |
| [Exam user interface](https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2/exam-user-interface/examui-performance-based-exams) | What the remote desktop, terminal and browser look like. Read it before exam day. |
| [PSI secure browser requirements](https://helpdesk.psionline.com/hc/en-gb/articles/4409608794260-PSI-secure-browser-and-Chrome-Extension-System-Requirements) | Check your machine well before the exam. |
| [Training portal](https://trainingportal.linuxfoundation.org/) | Scheduling, check-in, and access to the practice environment. |

## Documentation you can use during the exam

Practise navigating these; they're the only references you'll have.

- [kubernetes.io/docs](https://kubernetes.io/docs/home/). Bookmark-worthy pages:
  [kubectl quick reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/),
  [kubectl commands](https://kubernetes.io/docs/reference/kubectl/generated/),
  [JSONPath](https://kubernetes.io/docs/reference/kubectl/jsonpath/),
  [Kustomize task page](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [helm.sh/docs](https://helm.sh/docs/): the [CLI reference](https://helm.sh/docs/helm/helm/)
- [docs.podman.io](https://docs.podman.io/en/latest/Commands.html): only when a question links it

## Practice exams and question banks

| Resource | Notes |
|---|---|
| [killer.sh CKAD simulator](https://killer.sh/ckad) | Exam-style simulator; access is included when you buy the exam through the Linux Foundation. |
| [KodeKloud CKAD mock exam series](https://learn.kodekloud.com/user/courses/ultimate-certified-kubernetes-application-developer-ckad-mock-exam-series) | Several full mock exams with a live environment (paid). |
| [MyExamCloud CKAD practice tests](https://www.myexamcloud.com/onlineexam/ckad-free-practice-tests.course) | Free practice questions. |
| [dgkanatsios/CKAD-exercises](https://github.com/dgkanatsios/CKAD-exercises) | The classic free question bank, organised by curriculum area. |
| [jamesbuckett/ckad-questions](https://github.com/jamesbuckett/ckad-questions) | Walkthrough-style Q&A per curriculum domain. |
| [TiPunchLabs/ckad-dojo](https://github.com/TiPunchLabs/ckad-dojo) | Exam-style practice environment and questions. |
| [ExamCert CKAD practice questions and tips (2026)](https://www.examcert.app/blog/ckad-practice-questions-tips-2026/) | Practice questions with tips. |

## Topic deep dives

- **NetworkPolicy:** [ahmetb/kubernetes-network-policy-recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes).
  Numbered recipes (deny-all, allow from a namespace, egress-only, ...) with diagrams. The best way
  to build intuition for selector semantics. Also the
  [NetworkPolicy docs page](https://kubernetes.io/docs/concepts/services-networking/network-policies/).
- **Kustomize:** the [official examples](https://github.com/kubernetes-sigs/kustomize/tree/master/examples)
  and the [Kustomize reference](https://kubectl.docs.kubernetes.io/references/kustomize/). Useful
  for learning, but not allowed in the exam.
- **Podman:** [docs.podman.io command reference](https://docs.podman.io/en/latest/Commands.html).

## Write-ups from people who passed

- [How I passed CKAD in July 2025](https://medium.com/@keerthivikas/how-i-passed-ckad-in-july-2025-f07f6c213d75)
- [Notes for passing the CKAD exam and understanding the concepts of Kubernetes](https://medium.com/@shamre/notes-for-passing-the-ckad-exam-and-understanding-the-concepts-of-kubernetes-8de04720279b)
- [Passing the CKAD exam](https://talhajuikar.com/posts/passing-ckad-exam/)
- [Passing CKAD: cheatsheet, notes and tips](https://medium.com/@codebob75/passing-ckad-cheatsheet-notes-and-tips-1aa285e6a473)
- [I passed CKAD: what actually came up (after failing once)](https://codegenitor.medium.com/i-passed-ckad-heres-what-actually-came-up-and-what-helped-me-after-failing-once-fb5914b15f22)
- [The complete guide to the CKAD exam: passing in 4 attempts](https://medium.com/@sunwoopark/the-complete-guide-to-the-ckad-exam-how-to-pass-in-4-attempts-f2c7ef67ccab)
- [Failed CKAD first attempt: from 53 to 98 in a month](https://mengying-li.medium.com/failed-ckad-first-attempt-my-journey-to-improve-exam-score-from-53-to-98-in-a-month-part-2-f9e56f47cdf9)
- [How the CKAD certification changed from 2021 to 2024](https://itnext.io/how-the-ckad-certification-has-changed-from-2021-to-2024-06aec019a35a)
- [KodeKloud: CKAD exam verification guide](https://kodekloud.com/blog/ckad-exam-verification-guide/)
