# Le Pattern MVVM

## Introduction

Ce projet introduit le pattern de conception MVVM, ou Model-View-ViewModel. Il introduit :

- une architecture de base pour une application WPF utilisant MVVM
- la DB locale MS SQL Server Express (une alternative à SQLite)
- le chargement de données dans le code-behind d'un Control WPF indépendant, via l'ORM Entity Framework
- l'adaptation du code au pattern MVVM

## *Databinding* classique

Cette première partie sera très propre mais n'utilisera pas le pattern MVVM.

### `INotifyPropertyChanged`

- Créez un nouveau projet WPF (.NET Framework 5)
- Créez une classe `Common\Base.cs` (conservez le namespace proposé et ajoutez les importations nécessaires)

```csharp
/// <summary>
/// Cette classe implémente le pattern INotifyPropertyChanged
/// </summary>
public class Base : INotifyPropertyChanged
{
  /// <summary>
  /// L'événement PropertyChanged peut être lancé sur tout objet UI
  /// </summary>
  public event PropertyChangedEventHandler PropertyChanged;

  /// <summary>
  /// Lance l'événement sur la propriété passée
  /// (uniquement si le data binding est utilisé)
  /// </summary>
  /// <param name="nomPropriete">Le nom de la propriété
  /// qui est modifiée</param>
  protected void RaisePropertyChanged(string nomPropriete)
  {
    PropertyChangedEventHandler handler = this.PropertyChanged;
    // Lancer l'evt seulement si le handler est "connecté"
    if (handler != null)
    {
      PropertyChangedEventArgs args = new PropertyChangedEventArgs(nomPropriete);
      // Lance finlement l'evt PropertyChanged
      handler(this, args);
    }
  }
}
```

Cette classe servira de base à toute classe qui voudra « publier » certains types d'événements (c'est le but de l'interface `INotifyPropertyChanged` ici implémentée), du genre : « cette donnée a été modifiée, les entités concernées par cette modif pourront agir en conséquence ». Typiquement, l'action sera une mise à jour de l'UI, *bindée* (liée) à cette « donnée », qui sera en fait représentée par une **propriété C#** (rappel : sorte de variable exposée dans une classe avec getter/setter, dont on voit des exemples dans la section suivnte).

### BDD locale

- Créez un répertoire `BDD`
- Y créer un item de type `Service-based Database` (fichier `IntroMVVM.mdf`)
- Ouvrez le fichier mdf créé et, dans le Server Explorer, ouvrez une fenêtre de requête sur les tables
- Exécutez la requête T-SQL de création d'une table `[dbo].[Contact]` avec id (clé primaire auto-incrémentée), nom, email, tel
- Ajoutez quelques enregistrements à cette table (par requêtes ou en utilisant l'interface : `Show Table Data`)

### Entity Framework

Entity Framework est l'ORM de Microsoft, très utilisé dans les applications C#.

- Ajoutez au projet le package *EntityFramework* via NuGet
- Créez la classe `Entities\Contact.cs`

```csharp
[Table("Contact", Schema = "dbo")]
public class Contact : Base
{
  private int id;
  private string nom = string.Empty;
  private string email = string.Empty;
  private string tel = string.Empty;

  [Required]
  [Key]
  public int Id
  {
    get => id;
    set
    {
      id = value;
      RaisePropertyChanged("Id");
    }
  }

  [Required]
  public string Nom
  {
    get => nom;
    set
    {
      nom = value;
      RaisePropertyChanged("Nom");
    }
  }

  [Required]
  public string Email
  {
    get => email;
    set
    {
      email = value;
      RaisePropertyChanged("Email");
    }
  }

  [Required]
  public string Tel
  {
    get => tel;
    set
    {
      tel = value;
      RaisePropertyChanged("Tel");
    }
  }
}
```

Cette classe dérive de `Base`, dont on utilise `RaisePropertyChanged` : chaque propriété, quand elle est *set*, fait deux choses :
- l'affectation classique : `tel = value);` (rappel : `value` est la variable définie automatiquement dans un setter d'une propriété C# qui contient la nouvelle valeur) ;
- un appel à `RaisePropertyChanged` : on lance l'événement qui va avertir les abonnés éventuels que cette propriété a été modifiée. C'est la base du *binding* WPF, et du pattern MVVM.

### DbContext

On va s'occuper du `DbContext`, l'entité gérée par Entity Framework qui permet de faire le lien avec la couche données.

- Créez la classe `Models\ContactsDbContext.cs`
- Le nom passée au constructeur de la classe parente (appel à `base`) devra correspondre au nom de la future base de données construite, ici `IntroMVVM` ; prenez soin de respecter la casse quand vous définissez ces noms dans vos applis (comme c'est défini ici dans une String, le compilateur ne va pas vérifier si la DB existe)

```csharp
public class ContactsDbContext : DbContext
{
  public ContactsDbContext(DbContextOptions options) : base(options)
  {
  }

  public virtual DbSet<Contact> Contacts { get; set; }
}
```

- Notez la syntaxe de la propriété `Contacts` : getter/setter classique fournis sans code (générés automatiquement par C#)

### Connection String

- Ajoutez la Connection String dans le fichier de configuration `App.config`
- De nouveau, attention aux noms et à la casse

```xml
<connectionStrings>
  <add name="Contacts"
  connectionString="Server=(localdb)\MSSQLLocalDB;AttachDbFilename=|DataDirectory|Contacts.mdf;Database=Contacts;Trusted_Connection=Yes;"
  providerName="System.Data.SqlClient" />
</connectionStrings>
```

- Notez le `DataDirectory` entouré de caractèrees *pipes* : cela indique qu'il faut remplacer cette partie par une donnée fournie dans le code, qu'on va rendre disponible par les données du domaine de l'application (c'est une façon d'injecter des variables à des fichiers de configuration qui sont normalement complètement statiques)

### DataDirectory

Ce `DataDirectory` dans la *ConnectionString* précédente (en l'occurrence le répertoire `BDD`) sera donc une propriété de domaine qui illustre comment paramétrer, depuis l'application, un fichier de configuration. Elle est initialisée quand l'application démarre, dans le fichier `App.xaml.cs`, depuis la méthode `OnStartup` :

```csharp
protected override void OnStartup(StartupEventArgs e)
{
  base.OnStartup(e);

  // Le chemin construit ici va être plus tard injecté dans la Connection String à la place de |DataDirectory|
  // 
  string path = Environment.CurrentDirectory;
  path = path.Replace(@"\bin\Debug\net5.0-windows", "");
  path += @"\BDD\";
  AppDomain.CurrentDomain.SetData("DataDirectory", path);
}
```

- On va donc chercher le répertoire courant, auquel on retire le préfixe `\bin\debug` (en ajoutant `\net5.0-windows` si on est sur un projet .NET 5) pour « remonter » dans le répertoire principal du projet, puis on y ajoute `BDD` (c'est là que se trouve notre base de données MDF)

### Control indépendant pour la liste

- Ajoutez un item de type *User Control* nommé `ListControl` dans le répertoire `Controls`
- Le principe, dans une application classique avec une bonne expérience utilisateur, va être de :
  - définir un *User Control* pour chaque « écran » proposé à l'utilisateur
  - venir afficher le bon *User Control* dans la fenêtre principale en fonction des actions de l'utilisateur
- Notre appli ne contiendra que le *User Control* nouvellement créé
- Celui-ci contient une *ListView* (Control permettant d'afficher des donnée en mode liste, éventuellement en tableau mais par nécessairement) :

```xml
<ListView Name="lvContacts" ItemsSource="{Binding}">
    <ListView.View>
      <GridView>
        <GridViewColumn Header="ID" Width="Auto"
                        DisplayMemberBinding="{Binding Path=Id}" />
        <GridViewColumn Header="Nom" Width="Auto"
                        DisplayMemberBinding="{Binding Path=Nom}" />
        <GridViewColumn Header="Email" Width="Auto"
                        DisplayMemberBinding="{Binding Path=Email}" />
        <GridViewColumn Header="Tel" Width="Auto"
                        DisplayMemberBinding="{Binding Path=Tel}" />
      </GridView>
    </ListView.View>
  </ListView>
```

- On lui donne un nom (`lvContacts`) parce qu'on en a besoin pour y accéder depuis le *code-behind*
- En XAML, branchez l'événement `Loaded` du `UserControl` pour pouvoir y réagir dans le *code-behind*
  - rappel : le *code-behind* est le fichier `.xaml.cs` associé à tout fichier `.xaml` définissant une UI
  - on verra que l'utilisation de MVVM nous permettra de nous débarrasser de cette dépendance au *code-behind* pour charger les données
- Le code à ajouter au *code-behind* :

```csharp
private void UserControl_Loaded(object sender, RoutedEventArgs e)
{
  ChargerContacts();
}

private void ChargerContact()
{
  ContactsDbContext db;

  // Récupération de la Connection String depuis App.config
  string connString = ConfigurationManager.ConnectionStrings["Contacts"].ConnectionString;

  // Création des options de connexion pour Entity Framework
  var optionsBuilder = new DbContextOptionsBuilder<ContactsDbContext>();
  optionsBuilder.UseSqlServer(connString);

  try
  {
    // Connexion à la DB
    db = new ContactsDbContext(optionsBuilder.Options);
    
    // Récupération des contacts et peuplement du contexte de données de la ListView
    lvContacts.DataContext = db.Contacts.ToListAsync();
  }
  catch (Exception ex)
  {
    System.Diagnostics.Debug.WriteLine(ex.ToString());
  }
}
```

Ce code utilise Entity Framework pour récupérer les données depuis la table *Contact*. La méthode `ToListAsync` permet ensuite de récupérer une liste générique C# classique qui peut alors être liée à une `ListView` WPF, par l'intermédiare de la propriété `DataContext`.

Côté XAML, on a posé la propriété `ItemsSource` à `{Binding}`. Cela indique à `ListView` qu'elle sera nourrie dynamiquement à l'exécution (par son `DataContext`). Chaque enregistrement sera lié à une ligne de la liste dont le template est donné en XAML. Les attributs sont liés grâce à la propriété `DisplayMemberBinding`. Donc :
- La `ListView` est liée à la liste C# par le `DataContext`
- Chaque colonne de la `ListeView` est liée à une propriété des éléments de la liste C# sous-jacente (à savoir les propriétés exposées par `Contact` : `Nom`, `Email`...) par ce type de code XAML : `DisplayMemberBinding="{Binding Path=NomDeLaPropriétéDansLaClasseContact}"`

### MainWindow

- Peuplons un minimum la `Grid` de la fenêtre principale pour lancer le tout :
  - un `Menu`
  - une zone de contenu dans laquelle on va venir « coller » notre *User Control* grâce au *code-behind* quand l'utilisateur va cliquer sur `Contacts`

```xml
<Grid>
  <Grid.RowDefinitions>
    <RowDefinition Height="Auto" />
    <RowDefinition Height="*" />
  </Grid.RowDefinitions>
  <Menu Grid.Row="0" IsMainMenu="True">
    <MenuItem Header="_Fichier">
      <MenuItem Header="F_ermer" Click="MenuFermer_Click" />
    </MenuItem>
    <MenuItem Header="Contacts" Click="MenuContacts_Click" />
  </Menu>
  <Grid Grid.Row="1" Margin="10" HorizontalAlignment="Left"
        VerticalAlignment="Top" Name="zoneDeContenu" />
</Grid>
```

- Dans le *code-behind*, on réagit aux événements indiqués :

```csharp
private void MenuFermer_Click(object sender, RoutedEventArgs e)
{
  this.Close();
}

private void MenuContacts_Click(object sender, RoutedEventArgs e)
{
  // Ajout d'un élément enfant à la zone de contenu pour afficher notre ListControl
  zoneDeContenu.Children.Add(new ListControl());
}
```

### Test

Lancez l'application. Un clic sur *Contacts* devrait afficher la `ListView` avec les contacts précédemment insérés.

## Refactoring MVVM

Telle quelle, l'organisation du *binding* entre le modèle et la vue est déjà fonctionnelle.

On va cependant la refactorer pour implémenter le pattern MVVM qui va permettre un découplement optimal entre la vue et le modèle, grâce à la couche intermédiaire, le *ViewModel*. Cela améliorera grandement le maintenabilité de l'application à travers le temps, et permettra de tester l'application automatiquement (ce qui serait très difficile en l'état).

Et surtout, cela va aussi permettre d'avoir une gestion complètement transparente et indépendante du flux des données. Actuellement, si les données de la liste changent, la vue ne va pas automatiquement se mettre à jour (il faudra recharger la liste depuis le `DbContext` et réaffecter le `DataContext` de la `ListView` à chaque modification). Avec MVVM, la vue va « surveiller » toute modification du Model et cette mise à jour sera faite.

### ViewModelBase

- Créez la classe `Common\ViewModelBase.cs` qui étend les possibilités de `Base` (notification automatique). Dans notre application, elle n'ajoute aucune fonctionnalité, mais c'est une bonne pratique d'ajouter cette couche d'abstraction supplémentaire pour anticiper les besoins futurs.
- L'architecture étudiée par la suite va étendre quelque peu les fonctionnalités de cette classe.

```csharp
/// <summary>
/// Classe de base pour tous les View Models
/// </summary>
public class ViewModelBase : Base
{
}
```

### MVVM pour charger les contacts

Le code de chargement des contacts va être déplacé dans un View Model.

- Ajoutez une classe `ViewModels\ContactsViewModel.cs`

```csharp
public class ContactsViewModel : ViewModelBase
{
  private ObservableCollection<Contact> contacts = new ObservableCollection<Contact>();

  public ContactsViewModel() : base()
  {
    ChargerContacts();
  }

  public ObservableCollection<Contact> Contacts
  {
    get { return contacts; }
    set
    {
      contacts = value;
      RaisePropertyChanged("Contacts");
    }
  }
  public virtual void ChargerContacts()
  {
    ContactsDbContext db;
    try
    {
      db = new ContactsDbContext();
      Contacts = new ObservableCollection<Contact>(db.Contacts);
    }
    catch (Exception ex)
    {
      System.Diagnostics.Debug.WriteLine(ex.ToString());
    }
  }
}
```

- Supprimez les méthodes devenues superflues dans `Controls\ListControl.cs`

La classe `ObservableCollection` est très utilisée dans le pattern : elle lance automatiquement des notifications quand des éléments sont ajoutés ou supprimés de la collection, ou que toute la collection est actualisée. Les éléments WPF qui y sont liés sont alors immédiatement notifiés.

Notez à quel point la vue devient « propre » : pas de *code-behind*. Mais d coup, ce n'est pas tout à fait fini : il reste à « brancher » le View Model à cette vue pour qu'elle sache d'où vient la source du binding.

### Préparation du binding de la classe `ContactsViewModel`

N'importe quel *Control* ou fenêtre WPF peut créer, en XAML, des instances de classes. On va dire à notre vue d'instancier un View Model.

- Créez un namespace XML qu'on va appeler `vm` pour référencer le namespace du ViewModel (c'est comme un `using` en XAML) et créez une instance du View Model en lui affectant une clé de ressources, pour pouvoir la référencer :

```xml
<UserControl x:Class="WpfMVVMIntro.Controls.ListControl"
             ...
             xmlns:vm="clr-namespace:WpfMVVMIntro.ViewModels"
             ...>
```

- Supprimez l'attribut `Loaded` si ce n'est pas déjà fait (plus de *code-behind*)
- Supprimez l'attribut `Name` de la `ListView` (pas nécessaire mais on n'a plus besoin de la référencer dans le *code-behind*)

### Binding du ViewModel => View

On va utiliser la hiérarchie parent-enfant inhérente à XAML pour le data binding. Une pratique commune est de créer un View Model pour le lier à un User Control ou à une fenêtre WPF. On a donc « une View = un View Model ». Ensuite, dans la vue, on lie les Controls à des propriétés du View Model.

- Définir la ressource statique qui va instancier le View Model avec la clé `vmContacts` :

```xml
<UserControl.Resources>
  <vm:ContactsViewModel x:Key="vmContacts" />
</UserControl.Resources>
```

- Juste pour illustrer la transmission du contexte dans la hiérarchie XAML, encapsulez la `ListView` de `ListControl` dans une `Grid`, et appliquez un *contexte* à cette Grid, en utilisant la ressource statique précédemment définie :

```xml
<Grid DataContext="{StaticResource vmContacts}">
  <ListView ...
</Grid>
```

- La `ListView` va maintenant « se brancher » sur la propriété `Contacts` de ce contexte (définie dans le View Model) :

```xml
  <ListView ItemsSource="{Binding Path=Contacts}">
```

Souvenez-vous que c'est une `ObservableCollection` : tout changement de la liste sera immédiatement répercuté sur la vue, via ce binding.

### On résume

Le View Model peuple, via Entity Framework, une `ObservableCollection`. La vue référence le namespace du View Model, l'instancie en lui donnant une clé, utilise cette clé pour donner un contexte à la base de la hiérarchie XAML du User Control (la `Grid`). La `ListView` étant un enfant de la `Grid`, elle connaît ce contexte. Elle se lie alors à la propriété `Contacts` de ce contexte via sa propriété `ItemsSource`. Puis chaque « colonne » se lie à une propriété de la source (`Nom`, `Email`...). Le User Control n'a plus de *code-behind*, tout se passe en XAML. Le View Model est testable complètement indépendamment de la vue (et du Model). Les modifications sont répercutées automatiquement.

Tout revient donc à déplacer le *code-behind* qui assure les traitements des données dans une classe à part : un View Model. C'est la clé du pattern MVVM (avec les capacités de binding de XAML/.NET) : déplacer ce code dans une classe dédiée. Il ne faut pas se soucier d'être « 100 % sans *code-behind* ». C'est parfois impossible, une application typique va souvent avoir du code relatif à l'UI, et parfois même des appels au View Model. C'est tout à fait possible, l'important étant de respecter l'esprit du pattern : le binding automatique entre les données et la présentation. Cela renforce la réutilisabilité, la maintenabilité et la testabilité du code. De plus, le code est souvent plus lisible et le flux de contrôle de l'application plus facile à suivre.
