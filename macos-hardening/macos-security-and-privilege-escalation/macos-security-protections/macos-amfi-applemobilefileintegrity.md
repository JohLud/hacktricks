# macOS - AMFI - AppleMobileFileIntegrity

{% hint style="success" %}
Learn & practice AWS Hacking:<img src="../../../.gitbook/assets/arte.png" alt="" data-size="line">[**HackTricks Training AWS Red Team Expert (ARTE)**](https://training.hacktricks.xyz/courses/arte)<img src="../../../.gitbook/assets/arte.png" alt="" data-size="line">\
Learn & practice GCP Hacking: <img src="../../../.gitbook/assets/grte.png" alt="" data-size="line">[**HackTricks Training GCP Red Team Expert (GRTE)**<img src="../../../.gitbook/assets/grte.png" alt="" data-size="line">](https://training.hacktricks.xyz/courses/grte)

<details>

<summary>Support HackTricks</summary>

* Check the [**subscription plans**](https://github.com/sponsors/carlospolop)!
* **Join the** 💬 [**Discord group**](https://discord.gg/hRep4RUj7f) or the [**telegram group**](https://t.me/peass) or **follow** us on **Twitter** 🐦 [**@hacktricks\_live**](https://twitter.com/hacktricks\_live)**.**
* **Share hacking tricks by submitting PRs to the** [**HackTricks**](https://github.com/carlospolop/hacktricks) and [**HackTricks Cloud**](https://github.com/carlospolop/hacktricks-cloud) github repos.

</details>
{% endhint %}



## AppleMobileFileIntegrity.kext και amfid

Επικεντρώνεται στην επιβολή της ακεραιότητας του κώδικα που εκτελείται στο σύστημα παρέχοντας τη λογική πίσω από την επαλήθευση υπογραφής κώδικα του XNU. Είναι επίσης ικανό να ελέγχει τα δικαιώματα και να χειρίζεται άλλες ευαίσθητες εργασίες όπως η επιτρεπόμενη αποσφαλμάτωσης ή η απόκτηση θυρίδων εργασίας.

Επιπλέον, για ορισμένες λειτουργίες, το kext προτιμά να επικοινωνεί με τον daemon του χώρου χρήστη `/usr/libexec/amfid`. Αυτή η σχέση εμπιστοσύνης έχει καταχραστεί σε πολλές jailbreaks.

Το AMFI χρησιμοποιεί **MACF** πολιτικές και καταχωρεί τα hooks του τη στιγμή που ξεκινά. Επίσης, η αποτροπή της φόρτωσης ή της εκφόρτωσής του θα μπορούσε να προκαλέσει πανικό του πυρήνα. Ωστόσο, υπάρχουν ορισμένα επιχειρήματα εκκίνησης που επιτρέπουν την αποδυνάμωση του AMFI:

* `amfi_unrestricted_task_for_pid`: Επιτρέπει το task\_for\_pid να επιτρέπεται χωρίς απαιτούμενα δικαιώματα
* `amfi_allow_any_signature`: Επιτρέπει οποιαδήποτε υπογραφή κώδικα
* `cs_enforcement_disable`: Επιχείρημα σε επίπεδο συστήματος που χρησιμοποιείται για την απενεργοποίηση της επιβολής υπογραφής κώδικα
* `amfi_prevent_old_entitled_platform_binaries`: Ακυρώνει τα platform binaries με δικαιώματα
* `amfi_get_out_of_my_way`: Απενεργοποιεί εντελώς το amfi

Αυτές είναι μερικές από τις πολιτικές MACF που καταχωρεί:

* **`cred_check_label_update_execve:`** Η ενημέρωση ετικέτας θα εκτελείται και θα επιστρέφει 1
* **`cred_label_associate`**: Ενημερώνει την υποδοχή mac ετικέτας του AMFI με ετικέτα
* **`cred_label_destroy`**: Αφαιρεί την υποδοχή mac ετικέτας του AMFI
* **`cred_label_init`**: Μετακινεί 0 στην υποδοχή mac ετικέτας του AMFI
* **`cred_label_update_execve`:** Ελέγχει τα δικαιώματα της διαδικασίας για να δει αν θα πρέπει να επιτραπεί η τροποποίηση των ετικετών.
* **`file_check_mmap`:** Ελέγχει αν το mmap αποκτά μνήμη και την ορίζει ως εκτελέσιμη. Σε αυτή την περίπτωση, ελέγχει αν απαιτείται επικύρωση βιβλιοθήκης και αν ναι, καλεί τη λειτουργία επικύρωσης βιβλιοθήκης.
* **`file_check_library_validation`**: Καλεί τη λειτουργία επικύρωσης βιβλιοθήκης που ελέγχει μεταξύ άλλων αν ένα platform binary φορτώνει άλλο platform binary ή αν η διαδικασία και το νέο φορτωμένο αρχείο έχουν το ίδιο TeamID. Ορισμένα δικαιώματα θα επιτρέψουν επίσης τη φόρτωση οποιασδήποτε βιβλιοθήκης.
* **`policy_initbsd`**: Ρυθμίζει τις αξιόπιστες NVRAM κλειδιά
* **`policy_syscall`**: Ελέγχει τις πολιτικές DYLD όπως αν το binary έχει απεριόριστα τμήματα, αν θα πρέπει να επιτρέψει env vars... αυτό καλείται επίσης όταν μια διαδικασία ξεκινά μέσω `amfi_check_dyld_policy_self()`.
* **`proc_check_inherit_ipc_ports`**: Ελέγχει αν όταν μια διαδικασία εκτελεί ένα νέο binary άλλες διαδικασίες με δικαιώματα SEND πάνω από τη θυρίδα εργασίας της διαδικασίας θα πρέπει να τα διατηρήσουν ή όχι. Επιτρέπονται τα platform binaries, το δικαίωμα `get-task-allow` το επιτρέπει, τα δικαιώματα `task_for_pid-allow` επιτρέπονται και τα binaries με το ίδιο TeamID.
* **`proc_check_expose_task`**: επιβάλλει δικαιώματα
* **`amfi_exc_action_check_exception_send`**: Ένα μήνυμα εξαίρεσης αποστέλλεται στον αποσφαλματωτή
* **`amfi_exc_action_label_associate & amfi_exc_action_label_copy/populate & amfi_exc_action_label_destroy & amfi_exc_action_label_init & amfi_exc_action_label_update`**: Κύκλος ζωής ετικέτας κατά τη διάρκεια της διαχείρισης εξαιρέσεων (αποσφαλμάτωσης)
* **`proc_check_get_task`**: Ελέγχει τα δικαιώματα όπως το `get-task-allow` που επιτρέπει σε άλλες διαδικασίες να αποκτούν τη θυρίδα εργασίας και το `task_for_pid-allow`, που επιτρέπει στη διαδικασία να αποκτά τις θυρίδες εργασίας άλλων διαδικασιών. Αν κανένα από αυτά, καλεί το `amfid permitunrestricteddebugging` για να ελέγξει αν επιτρέπεται.
* **`proc_check_mprotect`**: Αρνείται αν το `mprotect` καλείται με τη σημαία `VM_PROT_TRUSTED` που υποδεικνύει ότι η περιοχή πρέπει να αντιμετωπίζεται σαν να έχει έγκυρη υπογραφή κώδικα.
* **`vnode_check_exec`**: Καλείται όταν εκτελέσιμα αρχεία φορτώνονται στη μνήμη και ορίζει `cs_hard | cs_kill` που θα σκοτώσει τη διαδικασία αν οποιαδήποτε από τις σελίδες γίνει μη έγκυρη
* **`vnode_check_getextattr`**: MacOS: Ελέγχει `com.apple.root.installed` και `isVnodeQuarantined()`
* **`vnode_check_setextattr`**: Όπως get + com.apple.private.allow-bless και εσωτερικό-ιστογράφημα-ισοδύναμο δικαίωμα
* &#x20;**`vnode_check_signature`**: Κώδικας που καλεί το XNU για να ελέγξει την υπογραφή κώδικα χρησιμοποιώντας δικαιώματα, cache εμπιστοσύνης και `amfid`
* &#x20;**`proc_check_run_cs_invalid`**: Παρεμβαίνει στις κλήσεις `ptrace()` (`PT_ATTACH` και `PT_TRACE_ME`). Ελέγχει για οποιαδήποτε από τα δικαιώματα `get-task-allow`, `run-invalid-allow` και `run-unsigned-code` και αν κανένα, ελέγχει αν η αποσφαλμάτωση επιτρέπεται.
* **`proc_check_map_anon`**: Αν το mmap καλείται με τη σημαία **`MAP_JIT`**, το AMFI θα ελέγξει για το δικαίωμα `dynamic-codesigning`.

`AMFI.kext` εκθέτει επίσης μια API για άλλες επεκτάσεις πυρήνα, και είναι δυνατό να βρείτε τις εξαρτήσεις του με:
```bash
kextstat | grep " 19 " | cut -c2-5,50- | cut -d '(' -f1
Executing: /usr/bin/kmutil showloaded
No variant specified, falling back to release
8   com.apple.kec.corecrypto
19   com.apple.driver.AppleMobileFileIntegrity
22   com.apple.security.sandbox
24   com.apple.AppleSystemPolicy
67   com.apple.iokit.IOUSBHostFamily
70   com.apple.driver.AppleUSBTDM
71   com.apple.driver.AppleSEPKeyStore
74   com.apple.iokit.EndpointSecurity
81   com.apple.iokit.IOUserEthernet
101   com.apple.iokit.IO80211Family
102   com.apple.driver.AppleBCMWLANCore
118   com.apple.driver.AppleEmbeddedUSBHost
134   com.apple.iokit.IOGPUFamily
135   com.apple.AGXG13X
137   com.apple.iokit.IOMobileGraphicsFamily
138   com.apple.iokit.IOMobileGraphicsFamily-DCP
162   com.apple.iokit.IONVMeFamily
```
## amfid

Αυτό είναι το daemon που εκτελείται σε λειτουργία χρήστη και θα χρησιμοποιήσει το `AMFI.kext` για να ελέγξει τις υπογραφές κώδικα σε λειτουργία χρήστη.\
Για να επικοινωνήσει το `AMFI.kext` με το daemon, χρησιμοποιεί mach μηνύματα μέσω της θύρας `HOST_AMFID_PORT`, η οποία είναι η ειδική θύρα `18`.

Σημειώστε ότι στο macOS δεν είναι πλέον δυνατό για τις διαδικασίες root να καταλάβουν ειδικές θύρες, καθώς προστατεύονται από το `SIP` και μόνο το launchd μπορεί να τις αποκτήσει. Στο iOS ελέγχεται ότι η διαδικασία που στέλνει την απάντηση έχει το CDHash σκληρά κωδικοποιημένο του `amfid`.

Είναι δυνατόν να δείτε πότε ζητείται από το `amfid` να ελέγξει ένα δυαδικό και την απάντησή του, αποσφαλματώνοντάς το και θέτοντας ένα breakpoint στο `mach_msg`.

Μόλις ληφθεί ένα μήνυμα μέσω της ειδικής θύρας, χρησιμοποιείται το **MIG** για να στείλει κάθε συνάρτηση στη συνάρτηση που καλεί. Οι κύριες συναρτήσεις έχουν αναστραφεί και εξηγηθεί μέσα στο βιβλίο.

## Provisioning Profiles

Ένα provisioning profile μπορεί να χρησιμοποιηθεί για να υπογράψει κώδικα. Υπάρχουν **Developer** προφίλ που μπορούν να χρησιμοποιηθούν για να υπογράψουν κώδικα και να τον δοκιμάσουν, και **Enterprise** προφίλ που μπορούν να χρησιμοποιηθούν σε όλες τις συσκευές.

Αφού υποβληθεί μια εφαρμογή στο Apple Store, αν εγκριθεί, υπογράφεται από την Apple και το provisioning profile δεν είναι πλέον απαραίτητο.

Ένα προφίλ συνήθως χρησιμοποιεί την επέκταση `.mobileprovision` ή `.provisionprofile` και μπορεί να απορριφθεί με:
```bash
openssl asn1parse -inform der -in /path/to/profile

# Or

security cms -D -i /path/to/profile
```
Αν και μερικές φορές αναφέρονται ως πιστοποιημένα, αυτά τα προφίλ παροχής έχουν περισσότερα από ένα πιστοποιητικό:

* **AppIDName:** Ο Αναγνωριστικός Κωδικός Εφαρμογής
* **AppleInternalProfile**: Δηλώνει ότι αυτό είναι ένα εσωτερικό προφίλ της Apple
* **ApplicationIdentifierPrefix**: Προστίθεται στο AppIDName (ίδιο με TeamIdentifier)
* **CreationDate**: Η ημερομηνία σε μορφή `YYYY-MM-DDTHH:mm:ssZ`
* **DeveloperCertificates**: Ένας πίνακας (συνήθως ενός) πιστοποιητικού/ων, κωδικοποιημένος ως δεδομένα Base64
* **Entitlements**: Τα δικαιώματα που επιτρέπονται με τα δικαιώματα για αυτό το προφίλ
* **ExpirationDate**: Η ημερομηνία λήξης σε μορφή `YYYY-MM-DDTHH:mm:ssZ`
* **Name**: Το Όνομα της Εφαρμογής, το ίδιο με το AppIDName
* **ProvisionedDevices**: Ένας πίνακας (για πιστοποιητικά προγραμματιστών) των UDIDs για τα οποία είναι έγκυρο αυτό το προφίλ
* **ProvisionsAllDevices**: Μια boolean (true για πιστοποιητικά επιχείρησης)
* **TeamIdentifier**: Ένας πίνακας (συνήθως ενός) αλφαριθμητικού συμβόλου/ων που χρησιμοποιούνται για να προσδιορίσουν τον προγραμματιστή για σκοπούς αλληλεπίδρασης μεταξύ εφαρμογών
* **TeamName**: Ένα αναγνώσιμο από άνθρωπο όνομα που χρησιμοποιείται για να προσδιορίσει τον προγραμματιστή
* **TimeToLive**: Η εγκυρότητα (σε ημέρες) του πιστοποιητικού
* **UUID**: Ένας Καθολικά Μοναδικός Αναγνωριστικός Κωδικός για αυτό το προφίλ
* **Version**: Αυτή τη στιγμή ορισμένο σε 1

Σημειώστε ότι η είσοδος δικαιωμάτων θα περιέχει ένα περιορισμένο σύνολο δικαιωμάτων και το προφίλ παροχής θα μπορεί μόνο να δώσει αυτά τα συγκεκριμένα δικαιώματα για να αποτρέψει την παροχή ιδιωτικών δικαιωμάτων της Apple.

Σημειώστε ότι τα προφίλ βρίσκονται συνήθως στο `/var/MobileDeviceProvisioningProfiles` και είναι δυνατή η έλεγχος τους με **`security cms -D -i /path/to/profile`**

## **libmis.dyld**

Αυτή είναι η εξωτερική βιβλιοθήκη που καλεί το `amfid` για να ρωτήσει αν θα πρέπει να επιτρέψει κάτι ή όχι. Αυτό έχει κακοποιηθεί ιστορικά στο jailbreaking τρέχοντας μια παραποιημένη έκδοση της που θα επέτρεπε τα πάντα.

Στο macOS αυτό βρίσκεται μέσα στο `MobileDevice.framework`.

## AMFI Trust Caches

Το AMFI του iOS διατηρεί μια λίστα γνωστών κατακερματισμών που είναι υπογεγραμμένοι ad-hoc, που ονομάζεται **Trust Cache** και βρίσκεται στην ενότητα `__TEXT.__const` του kext. Σημειώστε ότι σε πολύ συγκεκριμένες και ευαίσθητες λειτουργίες είναι δυνατή η επέκταση αυτού του Trust Cache με ένα εξωτερικό αρχείο.

## References

* [**\*OS Internals Volume III**](https://newosxbook.com/home.html)

{% hint style="success" %}
Learn & practice AWS Hacking:<img src="../../../.gitbook/assets/arte.png" alt="" data-size="line">[**HackTricks Training AWS Red Team Expert (ARTE)**](https://training.hacktricks.xyz/courses/arte)<img src="../../../.gitbook/assets/arte.png" alt="" data-size="line">\
Learn & practice GCP Hacking: <img src="../../../.gitbook/assets/grte.png" alt="" data-size="line">[**HackTricks Training GCP Red Team Expert (GRTE)**<img src="../../../.gitbook/assets/grte.png" alt="" data-size="line">](https://training.hacktricks.xyz/courses/grte)

<details>

<summary>Support HackTricks</summary>

* Check the [**subscription plans**](https://github.com/sponsors/carlospolop)!
* **Join the** 💬 [**Discord group**](https://discord.gg/hRep4RUj7f) or the [**telegram group**](https://t.me/peass) or **follow** us on **Twitter** 🐦 [**@hacktricks\_live**](https://twitter.com/hacktricks\_live)**.**
* **Share hacking tricks by submitting PRs to the** [**HackTricks**](https://github.com/carlospolop/hacktricks) and [**HackTricks Cloud**](https://github.com/carlospolop/hacktricks-cloud) github repos.

</details>
{% endhint %}