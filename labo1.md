# GUI with data layer and SOAP Web Service 

## Uitleg 



## Structuur

Todo: UML

3 Packages:
    - data
        * Woordenboek.java
        * WoordenboekDAO.java
    - gui 
        * WoordenboekFrame.form
        * WoordenboekFrame.java
    - impl
        * WoordenboekDAODummy.java


### Woordenboek.java

Niets speciaals, standaard classe met constructor en getters en setters.

### WoordenboekDAO.java

Niets speciaals, staandaard interface met 3 methodes die telkens een lijst teruggeven.

### WoordenboekFrame.form

XML van de frame (javax)

### WoordenboekFrame.java

Frame dat extends van javax.swing.JFrame.
Maakt in de constructor gebruik van de methode initComponents(). Hierin worden alle gui elementen gelinkt met de logica.
Verder wordt in de constructor een nieuwe woordenboekdummy gemaakt, men haalt de woordenboeken op en steekt men in de lijst woordenboeken.

Belangrijkste code:


```java

WoordenboekDAO woordenboekDAO;
    List<Woordenboek> woordenboeken;

    /**
     * Creates new form MainFrame
     */
    public WoordenboekFrame() {
        initComponents();
        try {
            woordenboekDAO = new WoordenboekDAODummy();
            woordenboeken = woordenboekDAO.getWoordenboeken();
            woordenboekLijst.setListData(new Vector(woordenboeken));
            if (woordenboeken.size() > 0) {
                woordenboekLijst.setSelectedIndex(0);
            }
            woordenLijst.setListData(new Vector<String>());
            definitieLijst.setListData(new Vector<String>());
        } catch (Exception e) {
            System.out.println("Applicatie kan niet opstraten:" + e.getMessage());
            throw new RuntimeException();
        }
    }

```

### WoordenboekDAODummy.java

"Belangrijkste classe". Implements van de interface.

```java

public class WoordenboekDAODummy implements WoordenboekDAO {

    List<Woordenboek> woordenboeken;
    Random random = new Random();

    public WoordenboekDAODummy() {
        this.woordenboeken = initWoordenboeken();
    }

    @Override
    public List<Woordenboek> getWoordenboeken() {
        return woordenboeken;
    }

    private List<Woordenboek> initWoordenboeken() {
        String[] ids = {"devils", "easton", "elements", "foldoc", "gazetteer",
            "gcide", "hitchcock", "jargon", "moby-thes", "vera", "wn", "world02"};
        String[] titels = {"THE DEVIL'S DICTIONARY ((C)1911 Released April 15 1993)",
            "Easton's 1897 Bible Dictionary",
            "Elements database 20001107",
            "The Free On-line Dictionary of Computing (27 SEP 03)",
            "U.S. Gazetteer (1990",
            "The Collaborative International Dictionary of English v.0.44",
            "Hitchcock's Bible Names Dictionary (late 1800's)",
            "Jargon File (4.3.1, 29 Jun 2001)",
            "Moby Thesaurus II by Grady Ward, 1.0",
            "Virtual Entity of Relevant Acronyms (Version 1.9, June 2002)",
            "WordNet (r) 2.0",
            "CIA World Factbook 2002"};
        List<Woordenboek> lijstWoordenboeken = new ArrayList<>();
        for (int i = 0; i < ids.length; i++) {
            lijstWoordenboeken.add(new Woordenboek(ids[i], titels[i]));
        }
        return lijstWoordenboeken;
    }

    @Override
    public List<String> zoekWoorden(String prefix, String woordenboekId) {
        List<String> woorden = new ArrayList<>();
        int aantal = random.nextInt(10);
        int startLetter = random.nextInt(26);
        for (int i = startLetter + 1 ; i < startLetter + aantal; i++) {
            woorden.add(prefix+woordenboekId+(char)i);
        }
        return woorden;
    }

    @Override
    public List<String> getDefinities(String woord, String woordenboekId) {
        List<String> definities = new ArrayList<>();
        int aantal = random.nextInt(10);
        for (int i = 1 ; i < aantal; i++) {
            definities.add("definitie"+i+woord+woordenboekId);
        }
        return definities;
    }
}

```

