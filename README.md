# asterisk-20.13.0-ms-teams-sbc
patch to support ms-teams and act like a sbc


The patch modifies Asterisk 20.17.0 so that Microsoft’s validation of the SBC (Session Border Controller) does not fail. The address in the SIP is rewritten to the value of
ms_signaling_address = ltd.domain.cy and must match the certificate. No SBC provider in Cyprus was found (for rent; the provider Primetel was also not ready yet as of end of 2025), and for latency reasons, none abroad was chosen (9ms within Cyprus and over 60ms to the mainland). With the patch, you can, for example, call with your Cypriot landline number in MS 365 / MS Teams…. You need the SIP data from your landline provider (in my case Primetel.com.cy; for Cyta, Epic, Kabel etc., you have to inquire, maybe they will provide it), configure Asterisk as a peer, and route calls through to MS Teams or run it as an extension.

Το patch τροποποιεί το Asterisk 20.17.0 ώστε η επικύρωση της Microsoft για το SBC (Session Border Controller) να μην αποτυγχάνει. Η διεύθυνση στο SIP αντικαθίσταται με την τιμή
ms_signaling_address = ltd.domain.cy και πρέπει να ταιριάζει με το πιστοποιητικό. Δεν βρέθηκε πάροχος SBC στην Κύπρο (για ενοικίαση, ο πάροχος Primetel δεν ήταν ακόμα έτοιμος (κατάσταση τέλος 2025)) και για λόγους καθυστέρησης δεν επιλέχθηκε κανένας στο εξωτερικό (9ms εντός Κύπρου και πάνω από 60ms προς την ηπειρωτική χώρα). Με το patch μπορείτε π.χ. να τηλεφωνείτε με τον κυπριακό σας σταθερό αριθμό στο MS 365 / MS Teams…. Χρειάζεστε τα SIP δεδομένα από τον πάροχο σταθερής σας τηλεφωνίας (στη δική μου περίπτωση Primetel.com.cy, για Cyta, Epic, Kabel κλπ πρέπει να ρωτήσετε, ίσως τα δώσουν), να ρυθμίσετε το Asterisk ως peer και να διαβιβάζετε κλήσεις στο MS Teams ή να το τρέχετε ως παράρτημα.
