# Analiza-filmy

>> Opis projektu

Projekt ten ma na celu przeprowadzenie szczegółowej analizy danych na temat wyprodukowanych filmów zawartych w bazie danych movies.csv, co pozwoli zapoznać się z preferencjami odbiorców oraz zwiększyć prawdopodobieństwo sukcesu przyszłych produkcji.

Analiza koncentruje się na:
- wyłonieniu reżyserów, którzy generują najwięsze zyski
- wyłonieniu aktorów, którzy generują najwięsze zyski
- wskazaniu najbardziej dochodowych, najlepiej ocenianych, najczęściej produkowanych filmów
- zbadaniu, co wpływa na sukces filmu

>> Opis danych i procesu przygotowania

I. Zastosowane narzędzia

- MS SQL - do przetwarzania i analizy danych 
- Python (Jupyter)
- Pandas – do przetwarzania danych
- Matplotlib – do tworzenia wykresów
- Seaborn – do wizualizacji danych

II. Dane

Wykorzystano ogólnodostępne dane movies.csv pobrane ze strony kaggle.com

III. Kroki analizy

- Wczytanie danych: dane zostały załadowane z pliku movies.csv do MS SQL
- Czyszczenie danych (konwersja wartości liczbowych na odpowiednie formaty, usuwanie tekstu z pól z oczekiwanymi wartościami liczbowymi), zapisanie danych w bazie Filmy.csv
- Wyłonienie reżyserów, którzy generują najwięsze zyski
- Wyłonienie najpopularniejszych aktorów
- Wskazanie najbardziej dochodowych, najlepiej ocenianych, najczęściej produkowanych filmów per całość i per gatunek
- Zbadanie korelacji budżetu, popularności, liczby głosów z dochodem z filmu

IV. Kroki analizy w kodzie SQL

1. Czyszczenie danych

       SELECT *
       FROM [Nauka1].[dbo].[Filmy]
       WHERE 
           TRY_CAST([budget] AS BIGINT) IS NULL AND [budget] IS NOT NULL OR
           TRY_CAST([popularity] AS FLOAT) IS NULL AND [popularity] IS NOT NULL OR
           TRY_CAST([revenue] AS BIGINT) IS NULL AND [revenue] IS NOT NULL OR
           TRY_CAST([vote_average] AS FLOAT) IS NULL AND [vote_average] IS NOT NULL OR
           TRY_CAST([vote_count] AS INT) IS NULL AND [vote_count] IS NOT NULL;
   
       UPDATE [Nauka1].[dbo].[Filmy]
       SET [budget] = NULL
       WHERE TRY_CAST([budget] AS BIGINT) IS NULL AND [budget] IS NOT NULL;

   Pokazuje, które dane zostały zaimportowane, jako tekst. Przy próbie konwersji danych na liczby pojawiały się błędy (kolumny zawierały tekst oraz daty).

   Wyrażenia z sekcji UPDATE powtórzono dla każdej kategorii.

   Błędne dane usunięto przypisując wartość NULL.

2. Wyłonienie reżyserów, którzy generują największe zyski

       SELECT TOP (20) [director], 
       	SUM(CAST([revenue] AS BIGINT)-CAST([budget] AS BIGINT)) AS [profit_sum], 
       	AVG(CAST([revenue] AS BIGINT)-CAST([budget] AS BIGINT)) AS [profit_avg]
       FROM [Nauka1].[dbo].[Filmy]
       GROUP BY [director]
       ORDER BY [profit_sum] DESC
       
       SELECT TOP (20) [director], 
       	SUM(CAST([revenue] AS BIGINT)-CAST([budget] AS BIGINT)) AS [profit_sum], 
       	AVG(CAST([revenue] AS BIGINT)-CAST([budget] AS BIGINT)) AS [profit_avg]
       FROM [Nauka1].[dbo].[Filmy]
       GROUP BY [director]
       ORDER BY [profit_avg] DESC

3. Przetwarzanie danych

       ALTER TABLE [Nauka1].[dbo].[Filmy]
       ADD [profit_sum] BIGINT;

   Rozszerza strukturę danych o kolumnę 'profit_sum'.

       UPDATE [Nauka1].[dbo].[Filmy]
       SET [profit_sum] = TRY_CAST([revenue] AS BIGINT) - TRY_CAST([budget] AS BIGINT)
       WHERE TRY_CAST([revenue] AS BIGINT) IS NOT NULL AND 
       		TRY_CAST([budget] AS BIGINT) IS NOT NULL;

   Uzupełnia kolumnę 'profit_sum' rozumianą, jako 'dochód = przychód - budzet'.

4. Wyłonienie najpopularniejszych aktorów

       SELECT
       	value AS dane_aktorow,
       	COUNT(*) AS liczba_wystapien,
       	SUM([profit_sum]) AS profit_razem
       FROM [Nauka1].[dbo].[Filmy]
       CROSS APPLY string_split([cast], ',')
       
       GROUP BY value
       ORDER BY liczba_wystapien DESC
       OFFSET 1 ROWS FETCH NEXT 20 ROWS ONLY;

   Pobiera grupy aktorów występujących w danych filmach i je rozdziela na poszczególne osoby. Zwraca dane w kolejności malejącej pod względem liczby wystapień aktorów w filmach.
   Omija pierwszy wiersz, w którym nie znajdowały się dane aktora i zwracane były błędne wartości.

5. Wskazanie najbardziej dochodowych, najlepiej ocenianych, najczęściej produkowanych filmów per całość i per gatunek

       SELECT TOP (20) 
       	[title],
       	[genres],
       	[vote_average],
       	[vote_count],
       	[profit_sum]
       FROM [Nauka1].[dbo].[Filmy]
       ORDER BY [profit_sum] DESC;
       
       SELECT TOP (10)
       	value AS genre,
       	COUNT(*) AS liczba_wystapien
       FROM [Nauka1].[dbo].[Filmy]
       CROSS APPLY string_split([genres], ' ')
       GROUP BY value
       ORDER BY liczba_wystapien DESC

   Zwraca 10 najczęściej występujących gatunków filmów uprzednio uzyskanych w wyniku rozdzielenia opisów w kolumnie 'genre' wraz z liczbą ich wystapień.

       SELECT TOP (20) 
       	[title],
       	[genres],
       	[vote_average],
       	[vote_count],
       	[profit] = [profit_sum]
       FROM [Nauka1].[dbo].[Filmy]
       WHERE 
       	[genres] LIKE '%Drama%'
       ORDER BY [profit] DESC;

   Zwraca 20 najbardziej dochodowych filmów z kategorii 'Drama'. Zapytanie powtórzono dla pozostałych uprzednio uzyskanych gatunków tj. Drama, Comedy, Thriller, Action, Romance, Adventure, Crime, Fiction, Science, Horror.

V. Kroki analizy w środowisku Python

1. Dostosowanie typu danych
   
       for kolumna in ['budget', 'revenue', 'profit_sum']:
           df[kolumna] = pd.to_numeric(df[kolumna], errors='coerce')
           df[kolumna] = df[kolumna].astype('Int64')

   Konwertuje wartości w kolumnach 'budget', 'revenue', 'profit_sum' na wartości numeryczne z pominięciem błędów i zmienia ich postać na liczby całkowite, żeby pozbyć się postaci wykładniczej liczb.

2. Wyłonienie reżyserów, którzy generują największe zyski

       top20_rezyserzy_profit_suma = df. groupby('director')['profit_sum'].sum()
       top20_rezyserzy_profit_suma = top20_rezyserzy_profit_suma.sort_values(ascending = False)
       top20_rezyserzy_profit_suma = top20_rezyserzy_profit_suma.reset_index()
       top20_rezyserzy_profit_suma = top20_rezyserzy_profit_suma.loc[:20,:]
       top20_rezyserzy_profit_suma

       plt.figure(layout = 'constrained', figsize=(10, 5))
       plt.title('TOP 20 reżyserów - dochody z filmów [mln]')
       rys1 = sns.barplot(x = 'director', y = 'profit_sum', data = top20_rezyserzy_profit_suma)
       
       for container in rys1.containers:
           for bar in container:
               height = bar.get_height()
               plt.text(
                   bar.get_x() + bar.get_width() / 2,
                   height,
                   f'{bar.get_height()/1e6:.1f}M',
                   ha = 'center',
                   va = 'top',
                   rotation=90
               )
               
       plt.xticks(rotation = 90)
       plt.savefig('rys1. Top 20 - reżyserzy i dochody z filmów.pdf', format = 'pdf')
       rys1.figure

   Tworzy wykres słupkowy TOP 20 reżyserów - dochody z filmów [mln] z odpowiednio sformatowanymi opisami danych (opisy 90 st., wartości w słupkach, jednostki w milionach) w celu zachowania lepszej przejrzystości.

       plt.close('all')
       
       x = np.arange(len(top20_rezyserzy_profit_avg))
       width = 0.3
       
       (rys3, ax) = plt.subplots(figsize = (12, 5))
       
       ax.set_title('TOP 20 reżyserów - dochody z filmów i średnie dochody z filmu [mln]')
       ax.bar(x - width / 2, top20_rezyserzy_profit_suma_avg['profit_suma'], width, label = 'dochod_suma')
       ax.bar(x + width / 2, top20_rezyserzy_profit_suma_avg['profit_avg'], width, label = 'dochod_avg')
       ax.set_xticklabels(top20_rezyserzy_profit_suma_avg['director'], rotation=90, ha='center')
       ax.set_xticks(x)
       
       for container in ax.containers:
           for bar in container:
               height = bar.get_height()
               plt.text(
                   bar.get_x() + bar.get_width() / 2,
                   height,
                   f'{bar.get_height()/1e6:.1f}M',
                   ha = 'center',
                   va = 'bottom',
                   rotation=90
               )
               
       plt.savefig('rys3. TOP20 - zagregowane, reżyserzy i dochody z filmów oraz średnie z filmu.pdf', format = 'pdf')
       rys3.figure

   Tworzy zagregowany wykres słupkowy TOP 20 reżyserów - dochody z filmów i średnie dochody z filmu [mln] z odpowiednio sformatowanymi opisami danych (opisy 90 st., wartości w słupkach, jednostki w milionach) w celu zachowania lepszej przejrzystości.

3. Wyłonienie aktorów, którzy generują najwięsze zyski

       nowy_df = df
       
       nowy_df['lista_aktorow'] = nowy_df['cast'].str.split(',')
       nowy_df_exploded = nowy_df.explode('lista_aktorow')
       
       top20_aktorzy_dochod = nowy_df_exploded.groupby('lista_aktorow').agg(
           liczba_filmow=('title', 'count'),
           profit_suma=('profit_sum', 'sum')
       ).sort_values(by='profit_suma', ascending=False)
       
       top20_aktorzy_dochod = top20_aktorzy_dochod.reset_index()
       
       top20_aktorzy_dochod = top20_aktorzy_dochod.loc[2:21,:]
       top20_aktorzy_dochod

   Wyrażenia:

       nowy_df['lista_aktorow'] = nowy_df['cast'].str.split(',')
       nowy_df_exploded = nowy_df.explode('lista_aktorow')
   tworzą listę osób z grupy aktorów występujących w formie STRING oddzielonych od siebie ' , ' oraz zapisują ją do nowej kolumny 'lista_aktorow'. Następnie funkcja EXPLODE rozbija tę listę na osobne wiersze.

   Wykres słupkowy przedstawiający Top 20 aktorów, którzy generują najwięsze zyski stworzony analogicznie, jak wcześniej.

4. Wskazanie najbardziej dochodowych, najlepiej ocenianych, najczęściej produkowanych filmów per całość i per gatunek

       df2 = df[['title', 'genres', 'vote_average', 'vote_count', 'profit_sum']]
       df2 = df2.sort_values('profit_sum', ascending = False)
       df2 = df2.reset_index()
       df2 = df2.loc[:19, :]
       df2

   Zwraca tabelę z określonymi danymi.

       df3 = df
       df3['gatunki'] = df3['genres'].str.split(" ")
       df3_exploded = df3.explode('gatunki')
       najczestsze_gatunki = df3_exploded.groupby('gatunki').agg(
           czestotliwosc_gatunkow = ('genres', 'count'),
           dochody = ('profit_sum', 'sum')
       ).sort_values('dochody', ascending = False)
       
       najczestsze_gatunki = najczestsze_gatunki.reset_index()
       najczestsze_gatunki = najczestsze_gatunki.loc[:10, :]
       najczestsze_gatunki = najczestsze_gatunki.drop(7)
       najczestsze_gatunki

   Tworzy tabelę z rozbitymi na poszczególne wiersze i zgrupowanymi gatunkami filmów, zestawia kolumnę 'gatunki' z liczbą ich wystapienia oraz ich łącznymi dochodami.

5. Automatyzacja filtrowania danych i prezentacji wyników dla wielu zapytań

       def filtruj_gatunki(df, gatunek):
           df_filtr = df[df['genres'].str.contains(gatunek, na = False, case = False)]
           df_filtr = df_filtr.sort_values('profit_sum', ascending = False)
           df_filtr = df_filtr[['title', 'genres', 'vote_average', 'vote_count', 'profit_sum']]
           df_filtr = df_filtr.reset_index()
           df_filtr = df_filtr.loc[:19 :]
    
           return df_filtr
   Tworzy funkcję, która filtruje dane po konkretnym gatunku.
   
       gatunki = najczestsze_gatunki['gatunki'].tolist()
       #gatunki
   Tworzy listę gatunków z poprzednio utworzonej kolumny 'gatunki'.
   
       for gatunek in gatunki:
           print(f'Filmy o największym dochodzie w kategorii {gatunek}')
           df_top = filtruj_gatunki(df5, gatunek)
           display(df_top)
   Iteruje przez listę gatunków i automatycznie wykonuje filtrowanie oraz prezentację wyników dla każdego gatunku.

6. Zbadanie korelacji budżetu, popularności, liczby głosów z dochodem z filmu

       df['popularity'] = pd.to_numeric(df['popularity'], errors='coerce')
       
       dane_niezalezne = ['budget', 'popularity', 'vote_count']
       dana_zalezna = ('profit_sum')
   
       plt.close('all')
       
       (figura, wykres) = plt.subplots(1, len(dane_niezalezne), figsize=(5 * len(dane_niezalezne), 5))
       
       for os, dana_n in zip(wykres, dane_niezalezne):
           sns.regplot(x=dana_n, y=dana_zalezna, data=df, ax=os)
           os.set_title(f'Regresja: {dana_n} vs {dana_zalezna}')
       
       plt.tight_layout()
       plt.savefig('rys4. Regresja - budżet, popularność, liczba głosów do dochodu.pdf', format = 'pdf')
       plt.show()
   Tworzy siatkę (figurę) n wykresów w sposób iteracyjny, gdzie n to długość listy 'dane_niezalezne'.

       from sklearn.linear_model import LinearRegression
       from sklearn.metrics import r2_score
   Korzysta z biblioteki sklearn do obliczenia:
- współczynników korelacji między budżetem, popularnością, liczba ocen a dochodem z filmu
- współczynnika wielowymiarowe regresji liniowej

       odmiana = ['budżetem', 'popularnością', 'liczbą ocen']
       for slowo, dana_n in zip (odmiana, dane_niezalezne):
           df_kor = df[[dana_n, dana_zalezna]].dropna()
           korelacja = df_kor[dana_n].corr(df_kor[dana_zalezna])
           print(f'Współczynnik korelacji między {slowo} i dochodem wynosi: {korelacja}')
           print('')
  Tworzy nowy DataFrame, z którego usuwa brakujące wartości i oblicza odpowiednie współczynniki korelacji.

       df_reg = df[['budget', 'popularity', 'vote_count', 'profit_sum']].dropna()

       model = LinearRegression()
       x = df_reg[['budget', 'popularity', 'vote_count']]
       y = df_reg['profit_sum']
       model.fit(x, y)
       y_pred = model.predict(x)
       r2 = r2_score(y, y_pred)
       
       print(f'Współczynnik regresji liniowej dla całego modelu R2 = {r2}')
Tworzy nowy DataFrame, z którego usuwa brakujące wartości i oblicza współczynnik wielowymiarowej regresji liniowej dla modelu (dane_niezalezne: 'budget', 'popularity', 'vote_count'], dana_zalezna: 'profit_sum').
   
>> Wizualizacje i analizy

W projekcie zostały wykorzystane różne metody wizualizacji wyników analizy:
  * Tabele zestawienione .csv, jako eksporty danych z MS SQL (grupa rozkłady.csv, top10.csv i top20.csv)
  * Tabele zestawieniowe z danymi zagregowanymi dotyczacymi reżyserów, dochodów, średnich przychodów, aktorów i częstotliwości ich udziału w produkcjach, tytułów, gatunków, rozkładu głosów
  * Wykresy słupkowe do przedstawienia rozkładu: 
    - rys1. Top 20 reżyserów w zależności od dochodu z filmów
    - rys2. Top 20 reżyserów w zależności od średniego dochodu z filmu
    - rys3. Top 20 reżyserów w zależności od dochodu z filmów wraz ze średnim dochodem z ich filmu
    - rys5. Top 10 najczęściej produkowanych filmów
    - rys6. TOP 20 aktorów generujących największe dochody z filmów
  * Wykresy liniowe
    - rys4. Korelacje budżetu, popularności, liczby głosów z dochodem z filmu
   
>> Wnioski
- Filmy łączące gatunki przygody i akcji dominują pod względem dochodów
- Szczegółowe zestawienie pokazuje, że filmy łączące gatunki akcji, przygody, science fiction, fantasy generują największe zyski
- Potwierdza to także zestawienie rezyserów, gdzie przede wszystkim Steven Spielberg, ale też i Peter Jackson oraz James Cameron to liderzy pod względem łącznych dochodów, co jest zgodne z ich dużymi hitami (np. "Avatar", "Władca Pierścieni", "Titanic")
- Dramaty czy romanse, choć są najczęstsze, mają mniejsze łączne dochody niż filmy akcji/przygody, co może wskazywać na większą ilość niskodochodowych produkcji dramatycznych (w kontekście dramaty/filmy akcji dramaty przy około dwukrotnie większej liczby filmów wygenerowały jedynie 70% dochodu, a to daje ok 30.5 mld mniej)
- Liczba filmów nie zawsze koreluje z wysokim dochodem z filmów — np. Bruce Willis, Sandra Bullock, czy Harrison Ford występowali często, ale dochody z ich filmów są niższe niż u aktorów z mniejszą liczbą wystąpień, ale za to z większymi hitami
- Uzyskane wykresy regresji liniowej pokazują, że wraz ze wzrostem budżetu filmu, jego popularności, liczby ocen dochód z filmu jest większy. Jednakże większe rozproszenie punktów, szczególnie w przypadku wykresu popularności filmu do jego dochodu, sugeruje słabsze dopasowanie i większą niepewność.

Współczynniki korelacji wynoszą odpowiednio: 
- między budżetem i dochodem wynosi: 0.559900785173437
- między popularnością i dochodem wynosi: 0.6203827032692386
- między liczbą ocen i dochodem wynosi: 0.7591854765550187

Współczynnik wielowymiarowej regresji liniowej dla całego modelu wynosi: R2 = 0.6000563904030516
