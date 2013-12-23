EntityFramework.RepositoryPattern
=================================

Lets observe two repositories with typical CRUD methods:

<pre>
public class NotificationsRepository : PPKC.DAL.BaseRepositories.INotificationsRepository 
    {
        public virtual List<Notification> GetAll()
        {
            using (var db = new SomeContext())
            {
                return db.Notifications.ToList();
            }
        }

        public virtual Notification Get(Guid id)
        {
            using (var db = new SomeContext())
            {
                return db.Notifications.FirstOrDefault(x => x.Id == id);
            }
        }

        public virtual Notification InserOrUpdate(Notification entity)
        {
            using (var db = new SomeContext())
            {

                db.Entry(entity).State = entity.Id == Guid.Empty || Get(entity.Id) == null ? EntityState.Added : EntityState.Modified;

                entity.CreatedAt = DateTime.UtcNow;
                entity.UpdatedAt = DateTime.UtcNow;

                db.SaveChanges();

                return entity;
            }
        }

        public virtual void Delete(Guid id)
        {
            using (var db = new SomeContext())
            {
                var entity = db.Notifications.FirstOrDefault(x => x.Id == id);
                if (null != entity)
                {
                    db.Notifications.Remove(entity);
                }
                db.SaveChanges();
            }
        }

        public virtual void Copy(Guid id)
        {
            using (var db = new SomeContext())
            {
                var dbSet = db.Notifications;
                var entityToCopy = dbSet.AsNoTracking().FirstOrDefault(x => x.Id == id);
                entityToCopy.Id = Guid.NewGuid();
                dbSet.Add(entityToCopy);
                db.SaveChanges();
            }
        }
    }
</pre>

and

<pre>
 public class ClientsRepository : IClientsRepository
    { 
        public virtual List<Client> GetAll()
        {
            using (var db = new SomeContext())
            {
                return db.Clients.ToList();
            }
        }

        public virtual Client Get(Guid id)
        {
            using (var db = new SomeContext())
            {                
                return db.Clients.FirstOrDefault(x => x.Id == id);
            }
        }

        public virtual Client InserOrUpdate(Client entity)
        {
            using (var db = new SomeContext())
            {

                db.Entry(entity).State = entity.Id == Guid.Empty || Get(entity.Id) == null ? EntityState.Added : EntityState.Modified;
                
                entity.CreatedAt = DateTime.UtcNow;
                entity.UpdatedAt = DateTime.UtcNow;

                db.SaveChanges();

                return entity;
            }
        }

        public virtual void Delete(Guid id)
        {
            using (var db = new SomeContext())
            {                
                var entity = db.Clients.FirstOrDefault(x => x.Id == id);
                if (null != entity)
                {
                    db.Clients.Remove(entity);                    
                }
                db.SaveChanges();
            }
        }

        public virtual void Copy(Guid id)
        {
            using (var db = new SomeContext())
            {
                var dbSet = db.Clients;
                var entityToCopy = dbSet.AsNoTracking().FirstOrDefault(x => x.Id == id);
                entityToCopy.Id = Guid.NewGuid();
                dbSet.Add(entityToCopy);
                db.SaveChanges();
            }
        }
    }
</pre>

The only different thing between the two is their DbSet: Clients/Notifications respectivelly. Since we can extract a DbSet via lambda lets move the code above to BaseRepository:

<pre>
public class BaseRepository<TEntity> : IBaseRepository<TEntity> where TEntity : class, IDBIdentity
    {
        Func<SomeContext, DbSet<TEntity>> ExtractDbSetFromContext { get; set; }

        public BaseRepository(Func<SomeContext, DbSet<TEntity>> extractDbSetFromContext)
        {
            this.ExtractDbSetFromContext = extractDbSetFromContext;
        }

        public virtual List<TEntity> GetAll()
        {
            using (var db = new SomeContext())
            {
                var dbSet = ExtractDbSetFromContext(db);
                return dbSet.ToList();
            }
        }

        public virtual TEntity Get(Guid id)
        {
            using (var db = new SomeContext())
            {
                var dbSet = ExtractDbSetFromContext(db);
                return dbSet.FirstOrDefault(x => x.Id == id);
            }
        }

        public virtual TEntity InserOrUpdate(TEntity entity)
        {
            using (var db = new SomeContext())
            {
                var dbSet = ExtractDbSetFromContext(db);

                db.Entry(entity).State = GetStateBasedOnIdentity(entity);

                EnsureIdentity(entity);

                entity.CreatedAt = DateTime.UtcNow;
                entity.UpdatedAt = DateTime.UtcNow;

                db.SaveChanges();

                return entity;
            }
        }

        private static void EnsureIdentity(TEntity entity)
        {
            if (entity.Id == Guid.Empty)
            {
                entity.Id = Guid.NewGuid();
            }
        }

        private EntityState GetStateBasedOnIdentity(TEntity entity)
        {
            return entity.Id == Guid.Empty || Get(entity.Id) == null ? EntityState.Added : EntityState.Modified;
        }

        public virtual void Delete(Guid id)
        {
            using (var db = new SomeContext())
            {
                var dbSet = ExtractDbSetFromContext(db);
                var entity = dbSet.FirstOrDefault(x => x.Id == id);
                if (null != entity)
                {
                    dbSet.Remove(entity);
                }
                db.SaveChanges();
            }
        }

        public virtual void Copy(Guid id)
        {
            using (var db = new SomeContext())
            {
                var dbSet = ExtractDbSetFromContext(db);
                var entityToCopy = dbSet.AsNoTracking().FirstOrDefault(x => x.Id == id);
                entityToCopy.Id = Guid.NewGuid();
                dbSet.Add(entityToCopy);
                db.SaveChanges();
            }
        }
    }
</pre>



now NotificationsRepository looks like this
<pre>
  public class NotificationsRepository : BaseRepository<Notification>, INotificationsRepository
    {
        public NotificationsRepository()
            : base(x => x.Notifications)
        {

        }

    }
    
    
    public interface INotificationsRepository : IBaseRepository<Notification>
    {
    }
</pre>

here is a common interface for 


