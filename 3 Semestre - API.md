# Portifólio das APIs - Marcos Vinicius Malaquias

Trabalho de graduação consistindo em um portfólio de projetos desenvolvidos durante as APIs: Aprendizado por Projeto Integrador; um projeto semestral interdisciplinar que tem como objetivo propor desafios junto a instiruições parceiras, simulando as necessidades do mercado e a jornada de um desenvolvedor no mundo atual. Este trabalho é apresentado à FATEC - Faculdade de Tecnologia de São José dos Campos como a entrega final para a obtenção da graduação em Banco de Dados.

## Um pouco sobre mim:

Meu nome é Marcos Vínicius Malaquias, sou entusiasta da programação e apaixonado por tecnologia. Estou cursando minha primeira graduação e escolhi o curso de Banco de Dados como uma forma de ingressar no mercado de tecnologia. 

Sou uma pessoa criativa, resiliente e analítica, escolhi a área de tecnologia devido à sua relevância na sociedade atual e também às projeções que indicam o crescimento contínuo dessa área nos próximos anos.

## Meus projetos

### Khali 3 - Aplicação Java Web para controle da jornada de trabalho

3º Semestre | 2023-2

Parceiro academico: `2RP`
Back-end: `Java`
Front-end: `React`
Banco de dados: `Postgres`

A aplicação Khali 3 é uma API Java web para o gerenciamento de horas extras e sobreavisos. Seu principal objetivo é controlar os apontamentos/solicitações de horas extras e sobreavisos realizados pelos colaboradores. Além disso, o sistema interpreta essas solicitações, direcionando-as aos responsáveis pela aprovação ou recusa. Caso aprovadas, o sistema realiza cálculos para determinar a quantidade de horas a serem pagas para cada colaborador, com base na parametrização configurada pelo setor de recursos humanos.

### Funcionalidades
- Telas para a criação de usuarios. centros de resultados e projetos;
- Fincionalidade para lançar apontamentos (Horas-extra/sobreaviso);
- Funcionalidade para a validação (Aprovar ou reprovar) apontamentos;
- Extração de relatórios de apontamentos;
- Dashboard a gestão dos apontamentos lançar e validações.

### Tecnologias utilizadas

#### Back-end: `Java`, `Spring-boot`, `Postgres`, `PG4Admin`, `Postman` e `Docker`
#### Front-end: `React` e `Typescrpt` 

### Contribuições Individuais

#### Sistema de notificações

Para que o software funcionasse de forma eficiente, era necessário a implementação de um sistema de notificações. Assim, sempre que um colaborador realizasse um apontamento, seu gestor seria notificado, e vice-versa: quando um gestor validasse um apontamento, o colaborador receberia uma notificação. Eu fiquei resposavél pelo desenvolvimento de todo esse sistema.

A solução que pensamos para essa funcionalidade era:

- Ao fazer uma solicitação, um novo registro era criado na tabela `notifications`, associando o seu apontamento ao seu gestor;

- Assim que seu gestor entra na aba de notificações ou na tela de aprovações, a notificação muda de status, assim a notificação ficará oculta pois o gestor já foi notificado;

- Assim que o gestor valida a solicitação, aprovando ou recusando, a tabela de notificações também é atualizada e a notificação ganha um novo status, mas dessa vez ela aparece para o solicitando, notificando que seu apontamento foi atualizado.

- Ao vizualizar a notificação, um gatilho deleta aquele registro no banco, assim poupando armazenamento.

<details>

<summary> Banco de Dados</summary>
<br>

```sql
DROP TABLE IF EXISTS notifications CASCADE;
CREATE TABLE IF NOT EXISTS notifications (
    apt_id INT PRIMARY KEY,
    usr_id integer,
    status boolean DEFAULT false,
    type apt_status DEFAULT 'Pending',
    CONSTRAINT fk_apt_id FOREIGN KEY
    (apt_id) REFERENCES appointments(apt_id),
    CONSTRAINT fk_usr_id FOREIGN KEY
    (usr_id) REFERENCES users(usr_id)
);
```
<br>

</details>

Optei por utilizar um banco relacional por conta do tenmpo disponivél para entregar essa tarefa e por já ter um domino com banco relacionais.

<details>

<summary> Entity </summary>
<br>

```java
@Entity
@Table(name = "notifications")
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
@EqualsAndHashCode
public class Notification {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "apt_id")
    private Long aptId;

    @ManyToOne
    @JoinColumn(name = "usr_id")
    private User userId;

    @Column(name = "status")
    private boolean status;

    @Enumerated(EnumType.STRING)
    @Column(name = "type")
    private AppointmentStatus type;

```
<br>

</details>

<details>

<summary> Service </summary>
<br>

```java
   @Query(value = "SELECT * FROM appointments a WHERE a.usr_id = :usr_id", nativeQuery = true)
    List<Appointment> findAppointmentByUser(@Param("usr_id") Long userId);

    @Query(value = "select * from appointments where rc_id in ( select rc_id from result_centers where gst_id = :usr_id) and status = 'Pending'", nativeQuery = true)
    List<Appointment> findByManager(@Param("usr_id") Long userId);

    @Query(value = "select * from appointments where rc_id in ( select rc_id from result_centers where gst_id = :usr_id)", nativeQuery = true)
    List<Appointment> findAllByManager(@Param("usr_id") Long userId);

    @Query(value = "update appointments set status = :#{#status.name()} where apt_id = :apt_id returning *", nativeQuery = true)
    Optional<Appointment> updateStatusAppointment(
        @Param("apt_id") Long apt_id,
        @Param("status") AppointmentStatus status
    );

    @Modifying
    @Query(value = "INSERT INTO notifications (apt_id, usr_id, type) VALUES (:aptId, :userId, 'Pending')", nativeQuery = true)
    void insertNotification(@Param("aptId") Long appointmentId, @Param("userId") Long userId);

    @Modifying
    @Query(value = "UPDATE notifications SET type = 'Rejected' WHERE apt_id = :aptId", nativeQuery = true)
    void updateToRejected(@Param("aptId") Long appointmentId);

    @Modifying
    @Query(value = "UPDATE notifications SET type = 'Approved' WHERE apt_id = :aptId", nativeQuery = true)
    void updateToApproved(@Param("aptId") Long appointmentId);

    @Modifying
    @Query(value =
        "UPDATE notifications " +
        "SET status = true " +
        "WHERE usr_id = :usr_id " +
        "AND type IN ('Rejected', 'Approved') " +
        "AND status = false",
        nativeQuery = true)
    void updateStatusToTrueForUser(@Param("usr_id") Long usr_id);

    @Query(value =
        "SELECT COUNT(*) FROM notifications n " +
        "WHERE n.apt_id IN (SELECT a.apt_id FROM appointments a " +
        "WHERE a.rc_id IN (SELECT rc.rc_id FROM result_centers rc WHERE rc.gst_id = :usr_id) " +
        "AND a.status = 'Pending')",
        nativeQuery = true)
    long countPendingNotificationsForManager(@Param("usr_id") Long userId);

    @Query(value =
        "SELECT COUNT(*) FROM notifications WHERE usr_id = :usr_id " +
        "AND status = false AND (type = 'Rejected' OR type = 'Approved')",
        nativeQuery = true)
    long countFalseRejectedOrApprovedNotifications(@Param("usr_id") Long userId);
```

<br>

</details>

<details>

<summary> Contreller </summary>
<br>

```java
  @Transactional
    @PostMapping
    public Appointment createAppointment(@RequestBody Appointment appointment) {
        Appointment savedAppointment = appointmentRepository.save(appointment);
        appointmentRepository.insertNotification(savedAppointment.getId(), savedAppointment.getUser().getId());
        return savedAppointment;
    }

    @Transactional
    @PutMapping("/{id}")
    public Appointment updateAppointment(@PathVariable Long id, @RequestBody Appointment appointmentDetails) {
        Appointment appointment = appointmentRepository.findById(id)

                .orElseThrow(() -> new EntityNotFoundException("Appointment not found with id: " + id));
        appointmentRepository.save(appointmentDetails);

        appointment.setApt_updt(appointmentDetails.getId());
        return appointmentRepository.save(appointment);
    }

    @Transactional
    @PutMapping("/validate/{id}")
    public Appointment updateAppointmentWithStatus(
            @PathVariable Long id,
            @RequestParam(name = "index") int index,
            @RequestParam(name = "feedback") String feedback) throws Exception {
        if (index != 1 && index != 2) {
            throw new Exception("O valor passado deve ser 1 ou 2");
        }
    
        Appointment appointment = appointmentRepository.findById(id)
                .orElseThrow(() -> new EntityNotFoundException("Appointment not found with id: " + id));
    
        AppointmentStatus status = AppointmentStatus.of(index);
        // appointment.setStatus(status);
        appointmentRepository.updateStatusAppointment(id, status);
        appointment.setFeedback(feedback);
        appointment = appointmentRepository.save(appointment);
    
        if (status == AppointmentStatus.Rejected) {
            appointmentRepository.updateToRejected(id);
        } else if (status == AppointmentStatus.Approved) {
            appointmentRepository.updateToApproved(id);
        }
        return appointment;
    }
    

    @GetMapping("/notification/{usr_id}")
    public List<Long> notificationAppointment(@PathVariable Long usr_id) {
        List<Long> notification = new ArrayList<>();
        
        long pendingNotificationsForManager = appointmentRepository.countPendingNotificationsForManager(usr_id);
        notification.add(pendingNotificationsForManager);
        
        long falseRejectedOrApprovedNotifications = appointmentRepository.countFalseRejectedOrApprovedNotifications(usr_id);
        notification.add(falseRejectedOrApprovedNotifications);
        
        return notification;
    }
    
    @Transactional
    @PutMapping("/notification/update/{usr_id}")
    public void updateNotificationsStatusToTrue(@PathVariable Long usr_id) {
        appointmentRepository.updateStatusToTrueForUser(usr_id);
    }
```

<br>

</details>

<details>

<summary> Componente notification ts </summary>
<br>

```ts
import { useState } from 'react';
import { Link } from 'react-router-dom';
import { NotificationItem } from '../services/AppointmentService';

interface NotificationPopUpProps {
    notificationItems: NotificationItem[];
    loadNotifications?: () => void;
}

export default function NotificationPopUp({ notificationItems, loadNotifications }: NotificationPopUpProps) {
    const [collapsed, setCollapsed] = useState(false);
    const [loaded, setLoaded] = useState(false);

    const toggleCollapsed = () => {
        setCollapsed(!collapsed);
        if (!loaded && loadNotifications) {
            loadNotifications();
            setLoaded(true);
        }
    };

    return (
        <div className={`notification ${notificationItems.length === 0 ? 'hidden' : ''}`}>
            <div>
                <button onClick={toggleCollapsed}>
                    {collapsed ? 'Expandir' : 'X'}
                </button>
                {!collapsed && notificationItems.length > 0 && (
                    <ul>
                        {notificationItems.map((item, index) => (
                            <li key={index}>
                                <Link to={item.url}>{item.label}</Link>
                            </li>
                        ))}
                    </ul>
                )}
            </div>
        </div>
    );
}
```

<br>

</details>

#### Dashboard gerencial

Desenvolvi o principal dashboard da aplicação, uma interface visual para a análise dos administradores, contendo todos os dados de todas as solicitações, status e responsáveis. Além disso, inclui filtros para facilitar a gestão e gráficos com alguns indicadores importantes.



<details>

<summary> Page dashboard </summary>
<br>

```ts
import "flatpickr/dist/themes/airbnb.css";
import { useEffect, useState } from 'react';
import BarChartDays from '../components/BarChartDaysOfMonth';
import BarChartHours from '../components/BarChartHoursOfDay';
import Filter from '../components/Filter';
import PieChart from '../components/PieChart';
import { AppointmentSchema } from '../schemas/Appointment';
import { UserSchema } from "../schemas/User";
import { getAppointmentsAdm } from '../services/AppointmentService';
import "../styles/dashboard.css";
import "../styles/filters.css";

interface AppointmentsProps {
    userLoggedIn: UserSchema;
}

export default function Appointments({ userLoggedIn }: AppointmentsProps) {
    const [appointments, setAppointments] = useState<AppointmentSchema[]>([]);
    const [filtered, setFiltered] = useState<AppointmentSchema[]>([]);

    const [filterValues, setFilterValues] = useState<{ [key: string]: any }>({
        "type": "",
        "status": "",
        "client": "",
        "resultCenter": "",
        "project": "",
        "startDate": "",
        "endDate": "",
    });

    const requestAppointments = () => {
        getAppointmentsAdm()
            .then(appointmentsResponse => {
                setAppointments(appointmentsResponse);
                applyFilters(filterValues, appointmentsResponse);
            });
    }

    useEffect(() => {
        requestAppointments();

    }, []);

    const handleFilterChange = (filterType: string, filterValue: any) => {
        // Para datas, garantimos que o formato esteja correto antes de definir no estado
        const formattedDate = filterValue instanceof Date ? filterValue.toLocaleDateString() : filterValue;

        const newFilterValues = { ...filterValues, [filterType]: formattedDate };
        setFilterValues(newFilterValues);
        applyFilters(newFilterValues, appointments);
        console.log(newFilterValues);
    };

    const applyFilters = (filters: { [key: string]: any }, data: AppointmentSchema[]) => {
        const newFiltered = data.filter((appointment) => {
            return Object.keys(filters).every((filterType) => {
                const filterValue = filters[filterType];
                if (filterValue === null || filterValue === undefined || filterValue === '') {
                    return true;
                }
                switch (filterType) {
                    case "type":
                        return appointment.type === filterValue;
                    case "status":
                        return appointment.status === filterValue;
                    case "client":
                        return appointment.client === filterValue;
                    case "resultCenter":
                        return appointment.resultCenter === filterValue;
                    case "project":
                        return appointment.project === filterValue;
                    case "startDate":
                        const filterStartDate = new Date(filterValue);
                        const appointmentStartDate = new Date(appointment.startDate);
                        /* Compare the dates without considering time components */
                        return appointmentStartDate.setHours(0, 0, 0, 0) >= filterStartDate.setHours(0, 0, 0, 0);
                    case "endDate":
                        const filterEndDate = new Date(filterValue);
                        const appointmentEndDate = new Date(appointment.endDate);
                        return appointmentEndDate.setHours(0, 0, 0, 0) <= filterEndDate.setHours(0, 0, 0, 0);
                    default:
                        return true;

                }
            });
        });

        // Atualiza o array filtrado
        setFiltered(newFiltered);
        console.log(newFiltered);
    };

    return (
        <div className="dashabord-admin-page">
            <div className="filters">
                <Filter
                    type="selection"
                    options={[
                        { label: 'Todos', value: '' },
                        { label: 'Hora Extra', value: 'Overtime' },
                        { label: 'Sobreaviso', value: 'OnNotice' },
                    ]}
                    onFilterChange={(value) => handleFilterChange("type", value)}
                />
                <Filter
                    type="selection"
                    options={[
                        { label: 'Todos', value: '' },
                        { label: 'Pendente', value: 'Pending' },
                        { label: 'Aprovados', value: 'Approved' },
                        { label: 'Recusados', value: 'Rejected' },
                    ]}
                    onFilterChange={(value) => handleFilterChange("status", value)}
                />
                <Filter
                    type="availableClients"
                    onFilterChange={(value) => handleFilterChange("client", value)}
                />
                <Filter
                    type="availableResultCenters"
                    onFilterChange={(value) => handleFilterChange("resultCenter", value)}
                    userLoggedIn={userLoggedIn}
                />
                <Filter
                    type="availableProjects"
                    onFilterChange={(value) => handleFilterChange("project", value)}
                />
                <Filter
                    type="date-start"
                    onFilterChange={(value) => handleFilterChange("startDate", value)}
                />
                <Filter
                    type="date-end"
                    onFilterChange={(value) => handleFilterChange("endDate", value)}
                />
            </div>
            <div className="charts-line1">
                <PieChart data={filtered} />
                <BarChartHours data={filtered} />
            </div>
                <div className="charts-line2">
                    <BarChartDays data={filtered} />
            </div>
            </div>
    );
}

```

<br>

</details>

#### Extração de relatórios

Nessa funcionalidade, o objetivo era disponibilizar ao usuário um arquivo CSV contendo todos os dados dos apontamentos, além disso, o usuário poderia filtrar as colunas para retornar apenas as desejadas. Fiquei responsável pela construção do serviço no front-end, que iria lidar com a requisição GET, passando como parâmetro as colunas desejadas.

<details>

<summary> Service jsx</summary>
<br>

```jsx
import axios from 'axios';

const API_URL = 'http://127.0.0.1:8080/csv-export';

export async function getReport(
    camposBoolean: boolean[],
    usrId: number
) {
    const camposBooleanString = camposBoolean.map(value => value.toString()).join(',');

    const params = {
        camposBoolean: camposBooleanString,
        usr_id: usrId,
    };

    try {
        const response = await axios.get(API_URL, { params });
        return response.data;
    } catch (error) {
        console.error('Erro na requisição:', error);
        throw error;
    }
}

```

<br>

</details>

##### Contruibuições adicionais

**Desenvolvimento Front-End**

- **Criação das Principais Telas da Aplicação:** Contribuí para o desenvolvimento das principais interfaces de usuário, garantindo uma experiência intuitiva e responsiva.
- **Criação dos Componentes:** Colaborei na estruturação e implementação de componentes reutilizáveis, promovendo a modularidade e a consistência visual.
- **Configuração das Rotas:** Ajudei na configuração das rotas dentro do componente principal (Home), assegurando a navegação fluida e a integração entre diferentes partes da aplicação.

**Desenvolvimento Back-End**

- **Criação dos Endpoints:** Contribuí para o desenvolvimento de endpoints RESTful, fornecendo acesso aos dados e funcionalidades da aplicação, e assegurando a correta mapeação de URLs para métodos controladores.
- **Implementação das Funções:** Participei da escrita de funções no back-end para manipular dados, realizar operações de CRUD e integrar com o banco de dados utilizando JPA.

**Documentação**

- **Responsável pela Construção do README:** Fui responsável pela elaboração da documentação do projeto no README, detalhando a configuração, uso, endpoints da API e outras informações essenciais para desenvolvedores e usuários finais.

**Outras Contribuições**

- **Modelagem do Banco de Dados:** Colaborei na modelagem do banco de dados, definindo entidades, relacionamentos e integridade referencial.
- **Criação de Entidades:** Participei do desenvolvimento de entidades no back-end, mapeando-as para tabelas do banco de dados e assegurando a correta persistência dos dados.
- **Otimização de Performance:** Contribuí para a identificação e implementação de otimizações de performance tanto no front-end quanto no back-end, melhorando a eficiência e a resposta da aplicação.
- **Implementação de Testes:** Ajudei no desenvolvimento de testes unitários e de integração para garantir a confiabilidade e a estabilidade da aplicação.
- **Suporte e Manutenção:** Ofereci suporte contínuo para a aplicação, corrigindo bugs, implementando novas funcionalidades e melhorando a experiência do usuário.

Essas contribuições foram fundamentais para o sucesso do projeto, garantindo uma aplicação robusta, eficiente e bem documentada.

#### Lições Aprendidas

Neste projeto, adquiri diversos aprendizados, especialmente por ser minha primeira experiência criando uma API Web utilizando Spring Boot e desenvolvendo o front-end em TypeScript.

**1. Desenvolvimento de API com Spring Boot:**  
- **Estrutura e Configuração do Projeto:** Aprendi a estruturar um projeto Spring Boot, configurando dependências e entendendo a hierarquia de pastas e arquivos.
- **Criação de Endpoints:** Compreendi como criar e gerenciar endpoints RESTful, mapeando URLs para métodos controladores.
- **Manipulação de Dados:** Utilizei JPA para interagir com o banco de dados, incluindo operações de CRUD (Create, Read, Update, Delete).
- **Protocolo HTTP:** Entendi profundamente o uso do protocolo HTTP para a transferência de informações, incluindo métodos como GET, POST, PUT e DELETE, além do tratamento de códigos de status HTTP.

**2. Desenvolvimento Front-End com TypeScript:**  
- **Integração com a API:** Aprendi a consumir a API Spring Boot no front-end, utilizando fetch e bibliotecas como Axios para realizar requisições HTTP.
- **Gerenciamento de Estado:** Utilizei ferramentas de gerenciamento de estado, como Redux ou Context API, para manter a consistência dos dados no front-end.
- **Componentização:** Criei componentes reutilizáveis em TypeScript, promovendo a modularidade e a manutenção do código.
- **Tratamento de Erros:** Implementei mecanismos de tratamento de erros para melhorar a experiência do usuário, exibindo mensagens de erro adequadas e lidando com falhas nas requisições.

**3. Boas Práticas e Padrões de Projeto:**  
- **Testes Unitários e de Integração:** Desenvolvi testes para garantir a funcionalidade e a confiabilidade da aplicação, utilizando frameworks como JUnit e Mockito para o back-end e Jest para o front-end.
- **Documentação e Comentários:** Aprendi a importância de documentar o código e as APIs, facilitando a manutenção e o entendimento por outros desenvolvedores.
- **Versionamento de Código:** Utilizei sistemas de controle de versão, como Git, para gerenciar as mudanças no código de forma eficiente e colaborativa.

Esses aprendizados foram fundamentais para o desenvolvimento do projeto e contribuíram significativamente para meu crescimento profissional como desenvolvedor full-stack.